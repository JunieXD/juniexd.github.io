---
title: 一次 FastAPI 异步应用性能优化问题
date: 2025-12-10
comments: true
author: Junie
---

这是一次关于 FastAPI 异步应用性能优化的真实记录。我在为 WordTower 项目开发「题目生成」功能时，遇到了一个经典的 **Event Loop 阻塞** 问题，导致即使命中了 Redis 缓存，接口响应依然会有 2 秒左右的延迟。

> [!question]- 什么是 Event Loop（事件循环）？
> Python 的 `asyncio` 是基于 **单线程** 的。可以把 Event Loop 想象成餐厅里 **唯一的服务员**。
> - **异步模式**：当服务员遇到需要等待的事情（比如客人看菜单、厨师做饭、等待网络请求返回），他不会傻站着等，而是记下来“这桌在等菜”，然后立刻转头去服务下一桌客人。这就是 `await` 的作用——交出控制权。
> - **同步阻塞**：如果你在代码里写了同步的、耗时的操作（比如复杂的 CPU 计算，或者同步的数据库查询），就像是一个客人拉着服务员聊天，或者让服务员亲自去厨房切菜。在服务员（Event Loop）做这件事的期间，**他无法响应任何其他客人的请求**。
> **结论**：在 FastAPI 中，只要主线程（Main Thread）被阻塞，整个服务器就“卡死”了，连简单的 `ping` 都响应不了。

很多 Python 开发者在使用 FastAPI 时容易陷入一个误区：以为加了 `async` 关键字，程序就自动“高性能”了。本文将复盘整个问题，并深入解释背后的 **Event Loop**、**线程池** 以及 **连接池** 原理。

## 1. 现象描述

我的需求很明确：
1. 用户请求 `/api/question/get` 获取题目。
2. 系统先查 Redis 缓存。
3. **Cache Hit**：如果有，直接快速返回（预期毫秒级）。
4. **Cache Miss**：如果没有，则现场调用 LLM 生成题目，存入数据库和 Redis，再返回（LLM 生成耗时约 5s）。
5. 每次返回题目后，后台会自动触发一个异步任务 (`BackgroundTasks`) 生成新的题目补充到队列中，以备下次使用。

**问题出现**： 我在测试时发现，明明 Redis 里已经预生成好了题目（日志显示 "Cache Hit!"），但 API 接口返回数据依然需要等待 **2 秒左右**。这完全不符合缓存命中的预期。

## 2. 问题排查

### 第一阶段：怀疑数据库查询
最初我怀疑是从 Redis 取出 ID 后，再去数据库查询完整题目内容的这个步骤慢了。
于是将逻辑改为：**直接把题目完整的 JSON 存入 Redis**，取出后直接返回，完全不查数据库。

- **结果**：日志显示 "Cache Hit!"，但响应时间依然是 2s。说明瓶颈不在数据库查询，而在其他地方。

### 第二阶段：锁定了阻塞源
我在代码中加入了详细的时间戳日志，发现通过 `asyncio.create_task` 启动的后台生成任务 (`generate_single_question`) 一旦开始执行，主线程似乎就被“卡住”了。

我仔细审查了 `generate_single_question` 函数的代码：

```python
# 这是一个 async 函数
async def generate_single_question(user_id: int):
    with DBSession(engine) as session:  # <--- 凶手在这里
        user = session.get(User, user_id)  # 同步阻塞 I/O
        # ...复杂的同步数据库查询逻辑...
        
        prompt = get_question_prompt(...)
        
        # 只有这里是真正的异步
        content = await generate_text(prompt) 
        
        # ...又是同步数据库写入逻辑...
        session.add(question)
        session.commit() # 同步阻塞 I/O
```

**根本原因分析**：
虽然函数定义为 `async`，但函数内部混杂了大量的 **同步阻塞代码**（SQLAlchemy 的同步 Session 操作）。
在 Python 的 `asyncio` 单线程模型中，**同步代码会阻塞 Event Loop**。这意味着当后台任务正在跑这些同步数据库操作时，整个主线程（包括处理 API 请求的逻辑）都会被挂起，直到这些同步操作执行完，主线程才有机会去响应那个已经准备好的 "Redis Cache Hit"。

## 3. 解决方案：彻底的异步化拆分

为了解决阻塞问题，需要将 **同步的 CPU/IO 密集型操作** 从主 Event Loop 中剥离出去，丢给**线程池**去跑。

> [!question]- 什么是线程池？
> 既然 Event Loop 只有一个（单线程），处理不了耗时的同步任务，那我们就找“外包团队”。
> **线程池** 就是预先创建好的一组后台线程（Workers）。 当 Event Loop 遇到同步的脏活累活（比如读写文件、同步数据库操作、图像处理）时，它可以使用 `asyncio.to_thread` 把这些任务扔给线程池里的某个线程去跑。
> - **主线程**：继续处理 HTTP 请求（高并发）。
> - **线程池**：在后台默默处理耗时的同步操作。

对 `generate_single_question` 进行重构：
### 第一步：拆分准备工作（Sync -> Thread Pool）
将所有生成前的数据库读取逻辑（查用户、算权重、选单词）提取到一个独立的同步函数 `_prepare_generation_data` 中。

```python
def _prepare_generation_data(user_id):
    # 纯同步的数据库操作
    with DBSession(engine) as session:
        # ... logic ...
        return q_type, word_ids, prompt
```

### 第二步：拆分写入工作（Sync -> Thread Pool）
将所有生成后的数据库写入逻辑（存题目、建立关联）提取到 `_save_generated_question` 中。

```python
def _save_generated_question(...):
    with DBSession(engine) as session:
        # ... save logic ...
        session.commit()
```

### 第三步：异步编排（Orchestration）
在主异步函数中，使用 `asyncio.to_thread` 将上述同步函数扔到线程池执行，只保留 LLM 调用在主线程 await。

```python
async def generate_single_question(user_id):
    # 1. 线程池跑同步读，不阻塞主线程
    data = await asyncio.to_thread(_prepare_generation_data, user_id)
    
    # 2. 纯异步 I/O，不阻塞
    content = await generate_text(data.prompt) 
    
    # 3. 线程池跑同步写，不阻塞主线程
    await asyncio.to_thread(_save_generated_question, ..., content)
```

## 4. 配套设施优化

为了配合这种高并发的执行模式，我们还做了两项基建升级：

1.  **全局 Redis 连接池**：Redis 的吞吐量极高，但如果每次 API 请求都新建 TCP 连接，性能会大打折扣。
2.  **扩大数据库连接池**：之前我们用的是同步的 SQLAlchemy，连接池默认很小。现在因为我们将大量数据库操作扔到了线程池中并发执行，意味着同一时刻会有多个线程尝试访问数据库，因此我们将 `pool_size` 调整到了 20，避免线程因为拿不到数据库连接而排队。

> [!question]- 什么是连接池？为什么要用它？
> 假设你要给数据库打电话查资料：
> - **没有连接池**：每次查资料，你都要先拨号、等待接通、验证身份（TCP 三次握手 + 数据库认证），查完后挂断。建立连接的过程极其缓慢（可能比查询本身还慢）。
> - **有连接池**：你和数据库之间保持着 10 通已经拨通的电话（连接池）。需要查资料时，直接拿起一个可用的听筒说话，说完放下听筒（归还连接），不挂断。

## 5. 最终效果

经过这番改造，**API 接口的响应速度从 2s 变成了瞬间返回（几十毫秒）**。
后台任务依然在勤勤恳恳地生成题目，但它们再也不会霸道地抢占主线程的资源了。

## 总结

在 FastAPI 中写 `async` 代码时，一定要时刻警惕：**不要在 async 函数里直接写耗时的同步 IO 操作（如同步的 ORM 查询）**。

一旦混入同步阻塞代码，`async` 就成了摆设。善用 `asyncio.to_thread` 将同步逻辑隔离到线程池，是解决此类问题的最佳实践。
