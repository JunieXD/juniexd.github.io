---
title: Vue Router 基础
date: 2025-11-07
comments: true
author: Junie
---
> Vue Router 官方文档：[入门 | Vue Router](https://router.vuejs.org/zh/guide/)

## 基础概念

### RouterLink 和 RouterView

Vue Router 提供了两个组件: `RouterLink` 和 `RouterView`。
#### RouterLink

`RouterLink` 是 Vue Router 提供的一个组件，用于在应用中生成一个可以点击的**导航链接**。它类似于 HTML 中的 `<a>` 标签，但它能阻止浏览器默认的完全页面跳转行为，利用 History API 进行**平滑切换**。

它可以自动识别当前 URL 是否与它指向的路径匹配，并自动添加一个特殊的 **“激活”类**，方便给当前页面链接添加样式（如高亮）。

#### RouterView

`RouterView` 是 Vue Router 中的一个**功能占位符组件**。它本身不渲染任何可见的 HTML 元素，它的作用是告诉 Vue ：“这里是**根据当前 URL 应该渲染的组件**的位置”。

当 URL 通过 `RouterLink` 发生变化时，`RouterView` 会根据路由配置表，销毁旧的组件，并在其位置上**动态渲染**与新 URL 匹配的**新组件**。
### router 和 route

| **概念**     | **含义**                            | **核心作用**                             | **状态**                             |
| ---------- | --------------------------------- | ------------------------------------ | ---------------------------------- |
| **router** | 路由管理器 (The Router Instance)       | 负责管理整个应用的**导航行为**、**路由规则**和**历史记录**。 | **全局的、不变的**配置对象和控制中心。              |
| **route**  | 当前路由信息 (The Current Route Object) | 一个对象，代表当前**激活的 URL 地址**和与之相关的所有信息。   | **动态的**，每次 URL 变化时，`route` 对象都会更新。 |

- **`router`** (管理器) 负责**做什么**（配置、跳转）。
- **`route`** (信息对象) 负责**是什么**（当前 URL 的状态）。

如果想要**改变页面**，使用 **`router`**。 如果想要**获取当前 URL 上的信息**（如参数），使用 **`route`**。

通过 `useRouter()` 和 `useRoute()` 函数创建 `router` 和 `route`。

```ts
const router = useRouter()
const route = useRoute()
```

### 创建路由器实例

通过调用 `createRouter()` 函数创建路由器实例。

```ts
import { createMemoryHistory, createRouter } from 'vue-router'

import HomeView from './HomeView.vue'
import AboutView from './AboutView.vue'

const routes = [
  { path: '/', component: HomeView },
  { path: '/about', component: AboutView },
]

const router = createRouter({
  history: createMemoryHistory(),
  routes,
})
```

## 路由匹配

### 带参数的动态路由匹配

```ts
const routes = [
  // 动态字段以冒号开始
  { path: '/users/:id', component: User },
]
```

可以通过 `route.params.id` 来访问。

### 捕获所有路由

```ts
const routes = [
  // 将匹配所有内容并将其放在 `route.params.pathMatch` 下
  { path: '/:pathMatch(.*)*', name: 'NotFound', component: NotFound },
]
```

可以用来做 404 NotFound 的页面。

### 正则表达式匹配

```ts
const routes = [
  // /:orderId -> 仅匹配数字
  { path: '/:orderId(\\d+)' },
  // /:productName -> 匹配其他任何内容
  { path: '/:productName' },
]
```

现在，转到 `/25` 将匹配 `/:orderId`，其他情况将会匹配 `/:productName`。`routes` 数组的顺序并不重要。

## 嵌套路由

```ts
const routes = [
  {
    path: '/user/:id',
    component: User,
    children: [
      {
        // 当 /user/:id/profile 匹配成功
        // UserProfile 将被渲染到 User 的 <router-view> 内部
        path: 'profile',
        component: UserProfile,
      },
      {
        // 当 /user/:id/posts 匹配成功
        // UserPosts 将被渲染到 User 的 <router-view> 内部
        path: 'posts',
        component: UserPosts,
      },
    ],
  },
]
```

子路由可以继承父路由的元数据。

```ts
meta: { requiresAuth: true }, // 父路由设置，所有子路由继承
```

## 命名路由

当创建一个路由时，我们可以选择给路由一个 `name`。然后我们可以使用 `name` 而不是 `path` 来传递 `to` 属性给 `<router-link>`。

```ts
<RouterLink :to="{ name: 'profile', params: { username: 'erina' } }" />
```

使用 `name` 有很多优点：
- 没有硬编码的 URL。
- `params` 的自动编码/解码。
- 防止你在 URL 中出现打字错误。
- 绕过路径排序，例如展示一个匹配相同路径但排序较低的路由。

所有路由的命名**都必须是唯一的**。如果为多条路由添加相同的命名，路由器只会保留最后那一条。

## 命名视图

如果想要同级展示多个视图，而不是嵌套展示，那就得用命名区分不同的视图，如果没命名，默认为`default`。

```ts
<router-view class="view left-sidebar" name="LeftSidebar" />
<router-view class="view main-content" />
<router-view class="view right-sidebar" name="RightSidebar" />

const router = createRouter({
  history: createWebHashHistory(),
  routes: [
    {
      path: '/',
      components: {
        default: Home,
        LeftSidebar: LeftSidebar,
        // 它们与 `<router-view>` 上的 `name` 属性匹配
        RightSidebar: RightSidebar
      },
    },
  ],
})
```

## 重定向和别名

### 重定向

```ts
const routes = [{ path: '/home', redirect: '/' }]
```

请注意，**[导航守卫](https://router.vuejs.org/zh/guide/advanced/navigation-guards.html)并没有应用在跳转路由上，而仅仅应用在其目标上**。在上面的例子中，在 `/home` 路由中添加 `beforeEnter` 守卫不会有任何效果。

### 别名

```ts
const routes = [{ path: '/', component: Homepage, alias: '/home' }]
```

通过别名，可以自由地将 UI 结构映射到一个任意的 URL，而不受配置的嵌套结构的限制。

用数组可以表示多个别名。

```ts
{ path: '', component: UserList, alias: ['/people', 'list'] },
```

## 导航守卫

### 全局前置守卫

```ts
const router = createRouter({ ... })

 router.beforeEach(async (to, from) => {
   if (
     // 检查用户是否已登录
     !isAuthenticated &&
     // ❗️ 避免无限重定向
     to.name !== 'Login'
   ) {
     // 将用户重定向到登录页面
     return { name: 'Login' }
   }
 })
```

> async 支持异步操作和返回 `Promise`，为了将来扩展性或考虑到 `isAuthenticated` 可能是一个异步检查的结果，开发者常常会默认加上 `async`。

> [!note]- Promise 是什么？
> `Promise` 是现代 **JavaScript** 中处理**异步操作**的核心机制，尤其是在处理网络请求、定时任务或复杂的导航逻辑时。
> 
> 我们可以把 `Promise` 理解为一个承诺（Promise）。当你执行一个异步操作（比如从服务器获取数据）时，函数不会立即返回最终结果，而是立即返回一个 `Promise` 对象。这个对象承诺在未来某个时间点，会给你一个结果（成功的数据）或一个失败的原因（错误）。
> > [!question]- 如何使用 Promise？
> > `Promise` 对象提供了 `.then()` 和 `.catch()` 方法来处理异步操作的最终结果。
> > `.then()`：处理成功的结果。
> > `.catch()`：处理失败的情况。

## 路由元信息

`meta` 属性可以将任意信息附加到路由上，并且它可以在路由地址和导航守卫上都被访问到。

```ts
const routes = [
  {
    path: '/posts',
    component: PostsLayout,
    children: [
      {
        path: 'new',
        component: PostsNew,
        // 只有经过身份验证的用户才能创建帖子
        meta: { requiresAuth: true },
      },
      {
        path: ':id',
        component: PostsDetail
        // 任何人都可以阅读文章
        meta: { requiresAuth: false },
      }
    ]
  }
]

router.beforeEach((to, from) => {
  // route.meta 方法是一个非递归合并所有 meta 字段（从父字段到子字段）的方法。
  // 不需要去检查每条路由记录
  // to.matched.some(record => record.meta.requiresAuth)
  if (to.meta.requiresAuth && !auth.isLoggedIn()) {
    // 此路由需要授权，请检查是否已登录
    // 如果没有，则重定向到登录页面
    return {
      path: '/login',
      // 保存我们所在的位置，以便以后再来
      query: { redirect: to.fullPath },
    }
  }
})
```

## 路由懒加载

路由懒加载即**当路由被访问的时候才加载对应组件**。

```ts
// 将
// import UserDetails from './views/UserDetails.vue'
// 替换成
const UserDetails = () => import('./views/UserDetails.vue')

const router = createRouter({
  // ...
  routes: [
    { path: '/users/:id', component: UserDetails }
    // 或在路由定义里直接使用它
    { path: '/users/:id', component: () => import('./views/UserDetails.vue') },
  ],
})
```