# Vue Router 4

> Vue Router 是 Vue 官方的路由管理器。本文从配置到守卫，从基础到实战。

---

## 一、路由是什么

### 1.1 本质

**URL 和组件的映射关系**。不同的 URL 渲染不同的组件，页面不刷新。

```
/about  →  About.vue
/users  →  Users.vue
/users/1 → UserDetail.vue
```

这本质上是前端接管了原本由后端处理的"根据 URL 返回内容"的职责，所以叫 SPA（Single Page Application）。

---

### 1.2 Hash vs History 模式

| | Hash 模式 | History 模式 |
|------|------|------|
| URL 样式 | `xxx.com/#/about` | `xxx.com/about` |
| 原理 | `hashchange` 事件 | `pushState` / `popState` |
| 服务端支持 | ✅ 不需要 | ⚠️ 需配置 fallback 到 index.html |
| SEO | ❌ `#` 后的内容不发送到服务端 | ✅ URL 干净 |
| 创建方式 | `createWebHashHistory()` | `createWebHistory()` |

```js
const router = createRouter({
  history: createWebHistory(),  // History 模式（生产环境需要服务端配置）
  // history: createWebHashHistory(),  // Hash 模式（简单，无需服务端配置）
  routes: [ /* ... */ ],
})
```

---

### 1.3 快速搭建

```bash
npm install vue-router@4
```

```js
// router/index.js
import { createRouter, createWebHistory } from 'vue-router'
import Home from '../views/Home.vue'
import About from '../views/About.vue'

const routes = [
  { path: '/', component: Home },
  { path: '/about', component: About },
]

const router = createRouter({
  history: createWebHistory(),
  routes,
})

export default router
```

```js
// main.js
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'

const app = createApp(App)
app.use(router)
app.mount('#app')
```

```vue
<!-- App.vue -->
<template>
  <nav>
    <router-link to="/">首页</router-link>
    <router-link to="/about">关于</router-link>
  </nav>

  <!-- 路由出口：匹配的组件渲染在这里 -->
  <router-view />
</template>
```

---

## 二、路由配置核心

### 2.1 动态路由

```js
const routes = [
  {
    path: '/users/:id',
    component: () => import('./views/UserDetail.vue'),
  },
  // 多个参数
  {
    path: '/posts/:category/:slug',
    component: () => import('./views/PostDetail.vue'),
  },
]
```

```vue
<!-- UserDetail.vue -->
<script setup>
import { useRoute } from 'vue-router'

const route = useRoute()
console.log(route.params.id)    // 用户 ID

// 或用 watch 监听参数变化
watch(() => route.params.id, async (newId) => {
  // 同一组件内，路由参数变化时重新加载数据
  const data = await fetchUser(newId)
  user.value = data
})
</script>
```

---

### 2.2 query 与 params

```js
// URL: /search?keyword=vue&page=2
const route = useRoute()
route.query.keyword  // 'vue'
route.query.page     // '2'

// URL: /users/42 —— 来自 /users/:id
route.params.id      // '42'
```

| | params | query |
|------|------|------|
| 定义方式 | 路由配置中 `:id` | URL 中 `?key=val` |
| URL 形式 | `/users/42` | `/search?q=vue` |
| 必填 | 通常是 | 可选 |
| 类型 | 默认字符串 | 默认字符串 |

---

### 2.3 嵌套路由

```js
const routes = [
  {
    path: '/user',
    component: () => import('./views/UserLayout.vue'),
    children: [
      // /user/profile
      { path: 'profile', component: () => import('./views/UserProfile.vue') },
      // /user/settings
      { path: 'settings', component: () => import('./views/UserSettings.vue') },
      // /user（默认子路由）
      { path: '', redirect: '/user/profile' },
    ],
  },
]
```

```vue
<!-- UserLayout.vue 中必须有嵌套的 <router-view> -->
<template>
  <div class="user-layout">
    <nav>
      <router-link to="/user/profile">资料</router-link>
      <router-link to="/user/settings">设置</router-link>
    </nav>

    <!-- 子路由匹配的组件渲染在这里 -->
    <router-view />
  </div>
</template>
```

---

### 2.4 命名路由

```js
const routes = [
  {
    path: '/users/:id',
    name: 'user-detail',    // ← 命名
    component: UserDetail,
  },
]
```

```vue
<!-- 跳转时用名字代替路径 -->
<router-link :to="{ name: 'user-detail', params: { id: 42 } }">
  用户详情
</router-link>
```

**好处**：路径变了只需改路由配置，所有用名字跳转的地方不用改。

---

### 2.5 路由传参（props 解耦）

```js
const routes = [
  // ① 布尔：params 作为 props 传入
  {
    path: '/users/:id',
    name: 'user',
    component: UserDetail,
    props: true,
  },
  // ② 对象：静态 props
  {
    path: '/promo',
    component: Promo,
    props: { isPromo: true },
  },
  // ③ 函数：动态处理
  {
    path: '/search',
    component: Search,
    props: (route) => ({ keyword: route.query.q }),
  },
]
```

```vue
<!-- UserDetail.vue：直接用 props，不依赖 useRoute -->
<script setup>
const props = defineProps({ id: String })
// props.id 就是路由参数，等价于 useRoute().params.id
</script>
```

**好处**：组件不依赖 `useRoute`，更易于测试和复用。

---

## 三、router-link

```vue
<template>
  <!-- 基本 -->
  <router-link to="/about">关于</router-link>

  <!-- 命名路由 -->
  <router-link :to="{ name: 'user', params: { id: 1 } }">用户</router-link>

  <!-- 带 query -->
  <router-link :to="{ path: '/search', query: { q: 'vue' } }">搜索</router-link>

  <!-- 替换当前历史记录（不产生新的历史条目） -->
  <router-link to="/about" replace />

  <!-- 自定义激活 class -->
  <router-link to="/about" active-class="active" />

  <!-- 自定义标签 -->
  <router-link to="/about" custom v-slot="{ navigate, isActive }">
    <button :class="{ active: isActive }" @click="navigate">
      关于
    </button>
  </router-link>
</template>
```

`router-link` 默认有两个激活 class：

| class | 匹配条件 |
|------|------|
| `.router-link-active` | 当前路由**以 to 的值开头** |
| `.router-link-exact-active` | 当前路由**精确匹配** to 的值 |

```
当前路由 /users/42
  to="/users"     → .router-link-active ✅  .router-link-exact-active ❌
  to="/users/42"  → 两个 class 都 ✅
```

---

## 四、编程式导航

```js
import { useRouter } from 'vue-router'

const router = useRouter()

// 字符串路径
router.push('/about')

// 命名路由
router.push({ name: 'user', params: { id: 1 } })

// 路径 + query
router.push({ path: '/search', query: { q: 'vue' } })

// 替换
router.replace('/login')

// 前进后退
router.go(1)   // 前进
router.go(-1)  // 后退
router.go(0)   // 刷新

// 注意 ⚠️：params 和 path 不能同时使用
router.push({ path: '/users', params: { id: 1 } })  // ❌ params 被忽略
router.push({ name: 'user', params: { id: 1 } })     // ✅ 用 name 代替
```

---

## 五、导航守卫（核心）

### 5.1 执行流程全景图

```
触发导航
  │
  ▼
离开的组件里调用 beforeRouteLeave       ← 组件内守卫
  │
  ▼
全局 beforeEach                        ← 全局守卫（最常用）
  │
  ▼
重用的组件里调用 beforeRouteUpdate      ← 组件内守卫
  │
  ▼
路由配置里调用 beforeEnter              ← 路由独享守卫
  │
  ▼
解析异步路由组件
  │
  ▼
被激活的组件里调用 beforeRouteEnter      ← 组件内守卫
  │
  ▼
全局 beforeResolve                      ← 全局守卫
  │
  ▼
导航确认
  │
  ▼
全局 afterEach                          ← 全局守卫（无 next，纯回调）
  │
  ▼
组件渲染，DOM 更新
```

---

### 5.2 全局守卫

```js
// router/index.js

// ① beforeEach —— 最常用：鉴权
router.beforeEach((to, from, next) => {
  // to：去哪里    from：从哪里来

  if (to.meta.requiresAuth && !isLoggedIn()) {
    next({ name: 'login', query: { redirect: to.fullPath } })  // 跳到登录页
  } else {
    next()  // 放行
  }
})

// ② beforeResolve —— 在确认导航前执行（所有组件内守卫和异步组件解析后）
router.beforeResolve((to, from, next) => {
  next()
})

// ③ afterEach —— 导航完成后（不能阻止导航，常用于页面标题、统计）
router.afterEach((to, from) => {
  document.title = to.meta.title ?? '默认标题'
  analytics.track('page_view', { page: to.fullPath })
})
```

---

### 5.3 路由独享守卫

```js
const routes = [
  {
    path: '/admin',
    component: Admin,
    beforeEnter: (to, from, next) => {
      if (currentUser.role === 'admin') {
        next()
      } else {
        next({ name: '403' })
      }
    },
  },
]
```

---

### 5.4 组件内守卫

```vue
<script setup>
import { onBeforeRouteLeave, onBeforeRouteUpdate } from 'vue-router'

// 离开当前路由时触发
onBeforeRouteLeave((to, from, next) => {
  if (hasUnsavedChanges.value) {
    const answer = confirm('你有未保存的修改，确定离开吗？')
    if (answer) {
      next()
    } else {
      next(false)  // 取消导航
    }
  } else {
    next()
  }
})

// 路由参数变化时触发（同一组件被复用时）
onBeforeRouteUpdate((to, from, next) => {
  // to.params.id 已经变了
  next()
})
</script>
```

**注意**：Vue Router 4 的组件内守卫用 Composition API 的方式是通过 `onBeforeRouteLeave` 和 `onBeforeRouteUpdate`。Options API 写法和之前一样（`beforeRouteEnter` 等）。

---

### 5.5 next 详解

```js
next()               // 放行，进入下一个钩子
next(false)          // 取消导航，URL 回退
next('/login')       // 重定向到 /login
next({ name: '404' })// 重定向到命名路由
next(error)          // 传入 Error 实例，导航被中止，错误被 router.onError() 捕获

// ⚠️ 不能多次调用 next()，也不要在 next() 后继续执行逻辑
```

---

## 六、路由元信息（meta）

```js
const routes = [
  {
    path: '/admin',
    component: Admin,
    meta: {
      requiresAuth: true,
      roles: ['admin'],
      title: '管理后台',
      keepAlive: true,
    },
  },
]
```

```js
// 在全局守卫中使用
router.beforeEach((to, from, next) => {
  // 鉴权
  if (to.meta.requiresAuth && !userStore.isLoggedIn) {
    return next('/login')
  }
  // 角色权限
  if (to.meta.roles && !to.meta.roles.includes(userStore.role)) {
    return next('/403')
  }
  next()
})

// 设置页面标题
router.afterEach((to) => {
  document.title = to.meta.title ?? '默认标题'
})
```

---

## 七、路由懒加载

```js
// ❌ 全部打包在一起，首屏加载慢
import Home from './views/Home.vue'
import About from './views/About.vue'
import Admin from './views/Admin.vue'

// ✅ 动态 import：按需加载，独立 chunk
const routes = [
  { path: '/', component: () => import('./views/Home.vue') },
  { path: '/about', component: () => import('./views/About.vue') },
  { path: '/admin', component: () => import('./views/Admin.vue') },
]
```

**按组分块**（Vite/Webpack 都支持魔法注释）：

```js
const routes = [
  {
    path: '/',
    component: () => import(/* webpackChunkName: "home" */ './views/Home.vue'),
  },
  {
    path: '/user',
    component: () => import(/* webpackChunkName: "user" */ './views/User.vue'),
    children: [
      {
        path: 'profile',
        component: () => import(/* webpackChunkName: "user" */ './views/UserProfile.vue'),
      },
    ],
  },
]
// Home 独立一个 chunk，User 相关的都打包到 user chunk
```

---

## 八、动态路由

### 8.1 添加路由

```js
// 运行时注册新路由（常用于后台权限路由）
router.addRoute({
  path: '/new-feature',
  component: () => import('./views/NewFeature.vue'),
})

// 添加到某个命名路由下（作为子路由）
router.addRoute('admin', {
  path: 'settings',
  component: () => import('./views/AdminSettings.vue'),
})
```

---

### 8.2 动态权限路由

```js
// router/index.js
export const staticRoutes = [
  { path: '/login', component: () => import('./views/Login.vue') },
  { path: '/404', component: () => import('./views/NotFound.vue') },
]

export const asyncRoutes = [
  {
    path: '/admin',
    component: () => import('./views/Admin.vue'),
    meta: { roles: ['admin'] },
  },
  {
    path: '/dashboard',
    component: () => import('./views/Dashboard.vue'),
    meta: { roles: ['admin', 'editor'] },
  },
]

// 创建 router 时只注册静态路由
const router = createRouter({
  history: createWebHistory(),
  routes: staticRoutes,
})
```

```js
// 登录后，根据用户角色动态注册路由
function addPermissionRoutes(role) {
  asyncRoutes
    .filter(route => route.meta.roles.includes(role))
    .forEach(route => router.addRoute(route))
}
```

---

### 8.3 删除路由

```js
// 按 name 删除
router.removeRoute('admin')

// 按 name 覆盖（addRoute 同名会覆盖旧的）
router.addRoute({ path: '/about', name: 'about', component: NewAbout })
```

---

## 九、滚动行为

```js
const router = createRouter({
  history: createWebHistory(),
  routes,
  scrollBehavior(to, from, savedPosition) {
    // 浏览器前进/后退 → 恢复之前的滚动位置
    if (savedPosition) {
      return savedPosition
    }

    // 有 hash → 滚动到对应锚点
    if (to.hash) {
      return { el: to.hash, behavior: 'smooth' }
    }

    // 其他情况 → 回到顶部
    return { top: 0 }
  },
})
```

---

## 十、组合式 API 全貌

```vue
<script setup>
import { useRoute, useRouter, onBeforeRouteLeave, onBeforeRouteUpdate } from 'vue-router'

// 当前路由信息（只读）
const route = useRoute()
route.path       // '/users/42'
route.params     // { id: '42' }
route.query      // { tab: 'profile' }
route.fullPath  // '/users/42?tab=profile'
route.meta       // { requiresAuth: true }
route.name       // 'user-detail'

// 路由实例（用于编程式导航）
const router = useRouter()
router.push({ name: 'home' })
router.replace('/login')
router.go(-1)
router.currentRoute  // 当前路由（响应式）

// 组件内守卫
onBeforeRouteLeave((to, from, next) => { /* ... */ })
onBeforeRouteUpdate((to, from, next) => { /* ... */ })
</script>
```

---

## 十一、实战：完整路由配置

```js
// router/index.js
import { createRouter, createWebHistory } from 'vue-router'
import { useUserStore } from '@/stores/user'

// 静态路由（所有人可访问）
const constantRoutes = [
  {
    path: '/login',
    name: 'login',
    component: () => import('@/views/Login.vue'),
    meta: { title: '登录' },
  },
  {
    path: '/404',
    name: '404',
    component: () => import('@/views/NotFound.vue'),
    meta: { title: '页面不存在' },
  },
]

// 动态路由（按需注册，根据权限过滤）
export const asyncRoutes = [
  {
    path: '/',
    component: () => import('@/layout/DefaultLayout.vue'),
    redirect: '/dashboard',
    children: [
      {
        path: 'dashboard',
        name: 'dashboard',
        component: () => import('@/views/Dashboard.vue'),
        meta: { title: '仪表盘', icon: 'chart', roles: ['admin', 'editor'] },
      },
      {
        path: 'users',
        name: 'users',
        component: () => import('@/views/Users.vue'),
        meta: { title: '用户管理', icon: 'user', roles: ['admin'] },
      },
      {
        path: 'settings',
        name: 'settings',
        component: () => import('@/views/Settings.vue'),
        meta: { title: '设置', icon: 'setting', roles: ['admin', 'editor'] },
      },
    ],
  },
  // 404 必须放最后
  { path: '/:pathMatch(.*)*', redirect: '/404' },
]

const router = createRouter({
  history: createWebHistory(),
  routes: constantRoutes,
  scrollBehavior: (to, from, savedPosition) => {
    return savedPosition || { top: 0 }
  },
})

// 全局前置守卫 —— 鉴权
router.beforeEach((to, from, next) => {
  const userStore = useUserStore()

  // 去登录页 → 直接放行
  if (to.name === 'login') {
    if (userStore.isLoggedIn) return next('/')  // 已登录 → 去首页
    return next()
  }

  // 未登录 → 去登录页
  if (!userStore.isLoggedIn) {
    return next({ name: 'login', query: { redirect: to.fullPath } })
  }

  next()
})

// 全局后置钩子 —— 标题和进度条
router.afterEach((to) => {
  document.title = to.meta.title ? `${to.meta.title} - Admin` : 'Admin'
})

export default router
```

---

## 十二、面试题

### 1. Hash 模式和 History 模式的原理和区别？

> - **Hash 模式**：监听 `hashchange` 事件，URL 中的 `#` 及其后面的内容变化不会向服务器发送请求，不需要服务端配置。
> - **History 模式**：使用 `pushState()` 和 `popstate` 事件，URL 和普通页面一样，刷新时会请求服务器，需要服务端配置 fallback 到 index.html（否则 404）。

---

### 2. 导航守卫的执行顺序？

> 1. `beforeRouteLeave`（离开组件）
> 2. `beforeEach`（全局前置）
> 3. `beforeRouteUpdate`（复用组件）
> 4. `beforeEnter`（路由独享）
> 5. `beforeRouteEnter`（进入组件）
> 6. `beforeResolve`（全局解析）
> 7. `afterEach`（全局后置，导航已确认）

---

### 3. 路由懒加载的原理？

> 使用动态 `import()`，Webpack/Vite 会将其拆分为独立的 chunk。只有当路由被访问时，对应的 chunk 才被下载并执行。这减少了首屏的 JS 体积，提升加载速度。

---

### 4. `$route` 和 `$router` 的区别？

> - `$route`（`useRoute()`）：当前路由信息对象（path、params、query、meta），是响应式的，但**只读**。
> - `$router`（`useRouter()`）：路由实例，用于**编程式导航**（push、replace、go、addRoute 等）。

---

### 5. 同一路由跳转时，如何让组件重新渲染？

> Vue 默认复用相同组件。方法：
> 1. 给 `<router-view>` 加 `:key="$route.fullPath"` — 强制重建
> 2. 在 `onBeforeRouteUpdate` 中处理参数变化
> 3. `watch(() => route.params.id, ...)` 手动响应

---

## 十三、总结

```
核心概念：
  router-link   → 声明式导航
  router-view   → 路由出口
  useRoute()    → 获取路由信息（只读）
  useRouter()   → 编程式导航

路由配置：
  动态路由 :id / 嵌套 children / 命名 name / props 解耦 / meta 元信息

导航守卫顺序（口诀）：
  离开 → 全局前 → 组复用 → 路由独享 → 组件进 → 全解析 → 全局后

实战要点：
  - 懒加载 all routes（动态 import）
  - meta.requiresAuth + beforeEach 做鉴权
  - 动态路由做权限控制
  - scrollBehavior 做滚动恢复
```

下一篇文章预告：**Pinia 状态管理** — 从安装到设计，替代 Vuex 的现代方案。
