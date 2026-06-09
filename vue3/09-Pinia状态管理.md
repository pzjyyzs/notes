# Pinia 状态管理

> Pinia 是 Vue 官方推荐的状态管理库，全面取代 Vuex。API 更简洁、TypeScript 支持更好、无需 mutation。

---

## 一、为什么需要状态管理

### 1.1 问题场景

```
App.vue
├── Header.vue          ← 需要 user.name
├── Sidebar.vue         ← 需要 user.role
└── Main.vue
    ├── UserProfile.vue ← 需要 user.name, user.role
    └── Settings.vue    ← 需要 user.preferences
```

**五个组件需要共享用户状态**。用 props 逐层传递会经过很多不需要这些数据的中间组件（Prop Drilling）。用 provide/inject 数据流不透明。这就是状态管理的用武之地。

---

### 1.2 Pinia vs Vuex

| | Pinia | Vuex 4 |
|------|------|------|
| **mutations** | ❌ 不需要 | ✅ 需要（繁琐） |
| **TypeScript** | ✅ 完美推断 | ⚠️ 需要额外类型声明 |
| **模块** | 天然模块化（多个 Store） | 嵌套 modules |
| **代码量** | 少 ~40% | 多 |
| **DevTools** | ✅ | ✅ |
| **官方推荐** | ✅ | ❌（已进入维护模式） |

---

## 二、快速上手

### 2.1 安装

```bash
npm install pinia
```

```js
// main.js
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'

const app = createApp(App)
app.use(createPinia())
app.mount('#app')
```

---

### 2.2 第一个 Store（Setup Store 风格，推荐）

```js
// stores/useCounterStore.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useCounterStore = defineStore('counter', () => {
  // ① state → ref / reactive
  const count = ref(0)

  // ② getters → computed
  const doubleCount = computed(() => count.value * 2)

  // ③ actions → 普通函数
  function increment() {
    count.value++
  }
  function decrement() {
    count.value--
  }
  function reset() {
    count.value = 0
  }

  // ④ 返回需要暴露的
  return { count, doubleCount, increment, decrement, reset }
})
```

```vue
<!-- 使用 -->
<script setup>
import { useCounterStore } from '@/stores/useCounterStore'

const counter = useCounterStore()
</script>

<template>
  <p>{{ counter.count }}</p>
  <p>双倍：{{ counter.doubleCount }}</p>
  <button @click="counter.increment()">+1</button>
</template>
```

---

### 2.3 Options Store 风格

```js
// stores/useCounterStore.js
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', {
  state: () => ({
    count: 0,
    name: '计数器',
  }),
  getters: {
    doubleCount: (state) => state.count * 2,
    // 使用其他 getter
    doublePlusOne(state) {
      return this.doubleCount + 1  // 注意：用 this 访问
    },
  },
  actions: {
    increment() {
      this.count++  // 用 this 访问 state
    },
    async fetchAndSet() {
      const data = await api.getCount()
      this.count = data  // 异步也没问题
    },
  },
})
```

**选择建议**：Setup Store 更像 Composition API，TypeScript 推断更好，是新项目的默认选择。

---

## 三、核心概念详解

### 3.1 State

```js
// Setup Store
import { defineStore } from 'pinia'
import { ref, reactive } from 'vue'

export const useUserStore = defineStore('user', () => {
  const user = ref({ name: 'Alice', role: 'admin' })
  const token = ref('')

  return { user, token }
})
```

**访问和修改**：

```vue
<script setup>
import { useUserStore } from '@/stores/useUserStore'

const userStore = useUserStore()

// 直接访问
console.log(userStore.user.name)

// 直接修改（不需要 mutation！）
userStore.user.name = 'Bob'

// 批量修改：$patch
userStore.$patch({
  token: 'new-token',
  user: { name: 'Bob', role: 'editor' },
})

// $patch 函数式（推荐用于复杂修改）
userStore.$patch((state) => {
  state.user.name = 'Bob'
  state.user.role = 'editor'
})

// 重置为初始状态
userStore.$reset()

// 替换整个 state（慎用）
userStore.$state = { user: {}, token: '' }
</script>
```

**结构化**：

```js
// ✅ 用 storeToRefs —— 只提取 state 和 getters，保持响应式
import { storeToRefs } from 'pinia'

const { user, token } = storeToRefs(userStore)
// user 和 token 是 ref，保持响应式

// ❌ 直接解构会丢失响应式
const { user, token } = userStore
// user 和 token 现在是普通对象/字符串，不再响应
```

> **规则**：state 和 getters 用 `storeToRefs` 解构；actions（方法）可以直接解构。

---

### 3.2 Getters

```js
export const useProductStore = defineStore('product', () => {
  const products = ref([...])
  const searchKeyword = ref('')

  // 基本 getter
  const total = computed(() => products.value.length)

  // 依赖其他 getter
  const inStock = computed(() => products.value.filter(p => p.stock > 0))
  const inStockCount = computed(() => inStock.value.length)

  // 带参数的 getter：返回函数
  const getById = computed(() => {
    return (id) => products.value.find(p => p.id === id)
  })

  // 搜索过滤（组合 state 和其他 getter）
  const filteredProducts = computed(() => {
    if (!searchKeyword.value) return products.value
    return products.value.filter(p =>
      p.name.includes(searchKeyword.value)
    )
  })

  return { products, total, inStock, inStockCount, getById, filteredProducts }
})
```

```vue
<script setup>
const store = useProductStore()

store.total              // 直接访问
store.getById(42)        // 带参数调用
store.filteredProducts   // 自动响应 searchKeyword 变化
</script>
```

---

### 3.3 Actions

```js
export const useAuthStore = defineStore('auth', () => {
  const user = ref(null)
  const loading = ref(false)
  const error = ref(null)

  // ✅ action 可以是异步的
  async function login(email, password) {
    loading.value = true
    error.value = null
    try {
      const res = await api.login(email, password)
      user.value = res.data.user
    } catch (e) {
      error.value = e.message
      throw e  // 抛出错误让调用方处理
    } finally {
      loading.value = false
    }
  }

  // ✅ action 可以调用其他 action
  function logout() {
    user.value = null
    // 不再需要手动清空 state
  }

  // ✅ 可以调用其他 store 的 action
  async function loginAndFetchData(email, password) {
    await login(email, password)
    const cartStore = useCartStore()
    await cartStore.fetchCart()  // 登录后拉取购物车
  }

  return { user, loading, error, login, logout }
})
```

**使用**：

```vue
<script setup>
const auth = useAuthStore()

async function handleLogin() {
  try {
    await auth.login(email, password)
    router.push('/dashboard')
  } catch (e) {
    alert('登录失败：' + e.message)
  }
}
</script>
```

---

## 四、订阅与监听

### 4.1 $subscribe — 监听 store 变化

```js
const store = useUserStore()

// 监听 store 中任何 state 变化
store.$subscribe((mutation, state) => {
  console.log('变化的 store：', mutation.storeId)
  console.log('变化后的 state：', state)
  // mutation.type: 'direct' | 'patch object' | 'patch function'
  // mutation.events: 变化的 key 数组（dev 模式）
})

// detached: true  → 组件卸载后订阅不被移除
store.$subscribe(callback, { detached: true })
```

**实用场景**：持久化到 localStorage。

```js
const store = useUserStore()

store.$subscribe((_, state) => {
  localStorage.setItem('user-store', JSON.stringify(state))
}, { detached: false })
```

---

### 4.2 $onAction — 监听 action 调用

```js
store.$onAction(({
  name,      // action 名称
  store,     // store 实例
  args,      // 传给 action 的参数
  after,     // action resolve 后回调
  onError,   // action reject 后回调
}) => {
  console.log(`action "${name}" 开始执行，参数：`, args)

  after((result) => {
    console.log(`action "${name}" 执行完成，返回值：`, result)
  })

  onError((error) => {
    console.error(`action "${name}" 报错：`, error)
  })
})
```

**实用场景**：全局 Loading、请求日志。

---

## 五、多个 Store 互相调用

```js
// stores/useAuthStore.js
export const useAuthStore = defineStore('auth', () => {
  const user = ref(null)

  async function login(email, password) {
    const res = await api.login(email, password)
    user.value = res.data.user
  }

  function logout() {
    user.value = null
  }

  return { user, login, logout }
})
```

```js
// stores/useCartStore.js
import { useAuthStore } from './useAuthStore'

export const useCartStore = defineStore('cart', () => {
  const items = ref([])

  function clearCart() {
    items.value = []
  }

  async function fetchCart() {
    const auth = useAuthStore()  // ← 直接在 action 中使用其他 store
    if (auth.user) {
      const res = await api.getCart(auth.user.id)
      items.value = res.data
    }
  }

  // 在 action 中响应其他 store 的变化
  function watchAuthAndFetch() {
    const auth = useAuthStore()
    watch(() => auth.user, (newUser) => {
      if (newUser) {
        fetchCart()
      } else {
        clearCart()
      }
    })
  }

  return { items, fetchCart, clearCart, watchAuthAndFetch }
})
```

> **注意**：不要在 store 顶层使用其他 store，放在 action 或者 getter 内部调用。因为在模块加载时，目标 store 可能还没创建。

---

## 六、插件

### 6.1 写一个插件

```js
// plugins/pinia-persist.js
// 持久化插件：自动把 store 的 state 存到 localStorage

export function createPersistPlugin(options = {}) {
  const { key = 'pinia-state' } = options

  return ({ store }) => {
    // 1. 初始化：从 localStorage 恢复
    const saved = localStorage.getItem(`${key}-${store.$id}`)
    if (saved) {
      store.$patch(JSON.parse(saved))
    }

    // 2. 每次 state 变化时保存
    store.$subscribe((_, state) => {
      localStorage.setItem(`${key}-${store.$id}`, JSON.stringify(state))
    })
  }
}
```

```js
// main.js
import { createPinia } from 'pinia'
import { createPersistPlugin } from './plugins/pinia-persist'

const pinia = createPinia()
pinia.use(createPersistPlugin({ key: 'my-app' }))

app.use(pinia)
```

---

### 6.2 常用插件

| 插件 | 功能 |
|------|------|
| `pinia-plugin-persistedstate` | 持久化到 localStorage/sessionStorage |
| `pinia-plugin-reset` | 给每个 store 添加 `$reset()` |
| `pinia-shared-state` | 多标签页/iframe 间同步状态 |

```bash
npm install pinia-plugin-persistedstate
```

```js
// main.js
import piniaPluginPersistedstate from 'pinia-plugin-persistedstate'

const pinia = createPinia()
pinia.use(piniaPluginPersistedstate)
```

```js
// 使用：在 store 中声明 persist
export const useUserStore = defineStore('user', () => {
  // ...
}, {
  persist: {
    key: 'user-store',
    storage: localStorage,  // 或 sessionStorage
    paths: ['token'],       // 只持久化 token
  },
})
```

---

## 七、实战：完整用户系统

```js
// stores/useAuthStore.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { api } from '@/api'

export const useAuthStore = defineStore('auth', () => {
  // ── State ──
  const user = ref(null)
  const token = ref(localStorage.getItem('token') || '')
  const loading = ref(false)
  const error = ref(null)

  // ── Getters ──
  const isLoggedIn = computed(() => !!token.value && !!user.value)
  const isAdmin = computed(() => user.value?.role === 'admin')
  const userName = computed(() => user.value?.name ?? '游客')

  // ── Actions ──
  async function login(email, password) {
    loading.value = true
    error.value = null
    try {
      const res = await api.post('/auth/login', { email, password })
      token.value = res.data.token
      user.value = res.data.user
      localStorage.setItem('token', res.data.token)
    } catch (e) {
      error.value = e.response?.data?.message || '登录失败'
      throw e
    } finally {
      loading.value = false
    }
  }

  async function fetchUser() {
    if (!token.value) return
    try {
      const res = await api.get('/auth/me')
      user.value = res.data
    } catch {
      // token 过期，清除
      logout()
    }
  }

  function logout() {
    user.value = null
    token.value = ''
    localStorage.removeItem('token')
  }

  return {
    user, token, loading, error,
    isLoggedIn, isAdmin, userName,
    login, logout, fetchUser,
  }
})
```

```vue
<!-- Login.vue -->
<script setup>
import { ref } from 'vue'
import { useAuthStore } from '@/stores/useAuthStore'
import { useRouter } from 'vue-router'

const auth = useAuthStore()
const router = useRouter()

const email = ref('')
const password = ref('')
const submitting = ref(false)

async function handleSubmit() {
  try {
    await auth.login(email.value, password.value)
    router.push('/dashboard')
  } catch (e) {
    alert(e.message)
  }
}
</script>
```

```vue
<!-- App.vue -->
<script setup>
import { useAuthStore } from '@/stores/useAuthStore'

const auth = useAuthStore()

// 应用启动时尝试恢复登录状态
auth.fetchUser()
</script>

<template>
  <template v-if="auth.isLoggedIn">
    <span>{{ auth.userName }}</span>
    <button @click="auth.logout()">退出</button>
  </template>
  <template v-else>
    <router-link to="/login">登录</router-link>
  </template>
</template>
```

---

## 八、Pinia vs provide/inject

| | Pinia | provide/inject |
|------|------|------|
| **适用场景** | 全局状态共享 | 局部状态共享（父组件到子孙） |
| **DevTools** | ✅ | ❌ |
| **插件生态** | ✅（持久化、同步等） | ❌ |
| **类型安全** | ✅ | ⚠️ 需要手动处理 |
| **数据流追踪** | 清晰 | 模糊（不知道谁 provide 的） |
| **调试** | Vue DevTools 完整支持 | 无法追踪 |

> **规则**：全局的、跨页面的用 Pinia；局部的、父到子孙的用 provide/inject。

---

## 九、目录结构建议

```
src/stores/
├── useAuthStore.js       # 认证相关
├── useUserStore.js       # 用户信息
├── useCartStore.js       # 购物车
├── useNotificationStore.js # 全局通知
├── useThemeStore.js      # 主题
└── index.js              # 可选：统一导出
```

---

## 十、面试题

### 1. Pinia 和 Vuex 的主要区别？

> 1. Pinia 没有 mutation，直接修改 state（通过 `$patch` 批量修改）
> 2. 完美的 TypeScript 支持，无需额外的类型声明
> 3. 天然模块化：每个 store 是一个独立实例，无需嵌套 modules
> 4. 更轻量：代码量减少约 40%
> 5. 支持 Composition API 风格的 Setup Store
> 6. 去掉了 `namespaced` 概念

---

### 2. storeToRefs 有什么用？为什么不直接解构？

> Pinia store 是一个 `reactive` 对象。直接解构会把 state 和 getters 变成普通值，丢失响应式。
> `storeToRefs` 将它们转为 `ref`，解构后的值仍保持响应式。
> 注意：`storeToRefs` **不会包含 actions**，因为 actions 是函数，不需要响应式。

```js
// ❌ 丢失响应式
const { count, doubleCount } = counterStore

// ✅ storeToRefs
const { count, doubleCount } = storeToRefs(counterStore)

// ✅ actions 直接解构就可以
const { increment } = counterStore
```

---

### 3. Pinia 怎么实现持久化？

> 1. 用 `$subscribe` 监听 state 变化 → 写入 localStorage
> 2. 创建 Pinia 实例时从 localStorage 恢复
> 3. 或者直接用 `pinia-plugin-persistedstate` 插件

---

### 4. 多个 store 互相引用时要注意什么？

> 不要在 store 顶层直接调用 `useXxxStore()`，因为模块加载顺序可能导致目标 store 还没创建。
> 应该在 action 或 getter 内部调用，确保在运行时（而非模块加载时）才访问其他 store。

---

### 5. `$patch` 和直接修改有什么区别？

> 直接修改：每次赋值都触发一次响应式更新。
> `$patch`：对同一个 tick 内的多次修改批量处理，只触发一次更新（类似 Vue 的批处理）。
> 此外 `$patch` 在 DevTools 中显示为一条日志，便于调试。

---

## 十一、总结

```
Pinia 核心结构：
  State    → ref / reactive
  Getters  → computed
  Actions  → 普通函数（可为 async）

关键 API：
  storeToRefs()      → 结构 state/getters 保持响应式
  $patch()           → 批量修改
  $reset()           → 重置为初始状态
  $subscribe()       → 监听 state 变化
  $onAction()        → 监听 action 调用

Setup Store > Options Store（推荐）
  与 Composition API 思维一致，TS 推断更好

最佳实践：
  - 按功能拆分 store（auth、user、cart）
  - store 之间在 action 中互相引用
  - 全局状态用 Pinia，局部状态用 provide/inject
```

下一篇预告：**编译优化原理** — Vue 3 为什么更快：静态提升、PatchFlags、Block Tree、靶向更新。
