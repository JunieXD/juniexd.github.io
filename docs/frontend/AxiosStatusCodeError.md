---
title: Axios 非 2xx 状态码抛异常问题
date: 2025-11-15
comments: true
author: Junie
---

> 如何优雅地处理 Axios 中的 HTTP 错误？
> 记一次被 Axios 抛异常导致逻辑判断中断的解决方案。

在前后端分离的项目中，我们经常使用 Axios 作为 HTTP 客户端。然而，许多开发者在处理业务逻辑错误（例如用户登录失败，返回 401 状态码）时，会发现一个令人困惑的现象：即使服务器返回了清晰的 JSON 错误信息，Axios 也会抛出异常，强制我们使用 `try...catch` 来处理本应属于正常业务流程的失败。

本文将深入分析这个问题，并提供两种优雅的解决方案。

## 问题的本质：HTTP 状态码与 Axios 的默认行为

### 1. 问题的表现

当我们的后端（如 FastAPI）在验证失败时返回 `400 Bad Request`、`401 Unauthorized` 或 `404 Not Found` 等状态码时，我们的前端调用代码：

```ts
// 示例：期望在这里判断 res.data.success
const res = await request.post('/api/auth/login', loginData); 
// 🚨 错误发生时，代码在这里中断，res 永远不会被赋值！
if (res.data.success) { /* ... */ }
```

会直接抛出 `AxiosError` 异常，使得 `res.data.success` 的判断逻辑完全失效。

### 2. 根本原因：Axios 的状态码设计

Axios 严格遵循 HTTP 协议约定，其核心设计逻辑是：

> 只要 HTTP 响应状态码**不在 2xx 范围内（即状态码 ≥ 300）**，Axios 就认为这是一个**请求层面的失败（Request Failure）**，并自动抛出异常。

即使服务器在 401 状态下返回了包含 `success: false` 的 JSON 数据，Axios 也会将整个响应对象封装进 `error.response` 属性中，并将请求转入错误处理流程。

## 解决方案：重写 Axios 默认行为

为了在业务逻辑层（例如 `login` 函数内部）处理这些非 2xx 状态码，我们需要在 Axios 的**响应拦截器**中进行干预。

#### 方案 A：全局使用 `validateStatus` (较少用，但更纯粹)

Axios 允许您配置 `validateStatus` 函数来决定哪些状态码应该被视为成功。

```ts
// 全局或局部配置
const api = axios.create({
    // 告诉 Axios：只要状态码小于 500，就不要抛出异常
    validateStatus: function (status) {
        return status < 500; // 允许 4xx 状态码进入 .then() 成功回调
    },
});
```

### 方案 B：在拦截器中“欺骗”Axios (推荐且更灵活)

这是最常用的方法。我们捕获错误，对于我们希望在业务逻辑中处理的错误（如 400, 401），我们将其状态码强制修改为 200，并返回这个修改后的响应对象。

**适用前提：** 后端（FastAPI）返回的 JSON 结构中必须包含一个业务状态字段（如 `success: boolean`）。

```ts
request.interceptors.response.use(
    (res) => {
        // 成功响应 (2xx)，直接返回
        return res;
    },
    async (err) => {
        if (err.response) {
            const status = err.response.status;

            // 针对业务错误（如登录失败 401，校验失败 400）进行处理
            if (status === 400 || status === 401 || status === 404) {
                // 1. 创建一个修改后的响应对象
                const modifiedResponse = {
                    ...err.response,
                    status: 200, // 关键：欺骗 Axios，使其进入 .then() 流程
                    statusText: 'OK',
                };
                
                // 2. 返回修改后的响应对象
                return modifiedResponse;
            }
            
            // 对于其他所有错误 (5xx 或未处理的 4xx)，继续抛出异常
            // 可以在这里统一处理全局的错误通知 (例如 500 错误提示)
        }
        
        // 返回原始错误，让 try...catch 或 .catch() 捕获
        return Promise.reject(err); 
    },
);
```

## 最终的业务逻辑

采用 **方案 B** 后，我们的登录函数将变得干净且专注于业务判断：

```ts
const login = async () => {
    // 拦截器保证这里总是能拿到 res 对象
    const res = await request.post('/api/auth/login', loginData); 
    
    // 业务代码只需判断后端 JSON 中的 success 字段
    if (res.data.success) {
      // 登录成功逻辑
    } else {
      // 登录失败逻辑，res.data.message 中是错误信息
      showNotification('登录失败', res.data.message);
    }
}
```

通过这种方式，我们将 HTTP 状态码的处理逻辑隔离到了 Axios 拦截器中，让应用层代码更加简洁、可读，并更符合业务逻辑的判断习惯。