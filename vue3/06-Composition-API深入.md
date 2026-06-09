# Vue 3 Composition API 深入

> 组合式 API 不只是另一种写法，而是一种全新的代码组织方式。本文从思维转变到实战模式，覆盖逻辑复用的全部技巧。

---

## 一、为什么要 Composition API

### 1.1 Options API 的痛点

```js
// Options API —— 同一功能的代码散落在不同选项中
export default {
  data() {
    return {
      keyword: '',          // 搜索功能的数据
      results: [],
      loading: false,
      form: { name: '' },   // 表单功能的数据
      errors: {},
    }
  },
  watch: {
    keyword: {              // 搜索功能的侦听 —— 离 data 很远
      handler: 'search',
    }
  },
  methods: {
    search() { /* ... */ },          // 搜索
    clearSearch() { /* ... */ },     // 搜索
    validateForm() { /* ... */ },    // 表单 —— 和搜索混在一起
    submitForm() { /* ... */ },      // 表单
  }
}
```

问题很明显：**同一功能的相关代码被 `data`、`methods`、`watch` 强行拆散**，功能多了以后就像在玻璃渣里找东西。

---

### 1.2 Composition API 的解决方案

```vue
<script setup>
import { ref } from 'vue'

// 🟢 搜索功能 —— 所有相关代码聚在一起
const { keyword, results, loading, search } = useSearch()

// 🔵 表单功能 —— 所有相关代码聚在一起
const { form, errors, validate, submit } = useForm()

// 它们互不干扰，井然有序
</script>
```

**核心思想**：不再按「选项类型」组织代码，而是按「功能关注点」组织代码。

---

### 1.3 对比

```
Options API：               Composition API：
┌──────────────┐            ┌──────────────┐
│ data         │            │ useSearch()  │  ← 搜索的所有东西
│   keyword    │            │   数据结构    │
│   results    │            │   侦听        │
│   form       │            │   方法        │
├──────────────┤            ├──────────────┤
│ methods      │            │ useForm()    │  ← 表单的所有东西
│   search()   │            │   数据结构    │
│   clear()    │            │   校验        │
│   validate() │            │   提交        │
│   submit()   │            ├──────────────┤
├──────────────┤            │ useTheme()   │  ← 主题
│ watch        │            └──────────────┘
│   keyword    │
└──────────────┘

按「选项类型」组织          按「功能关注点」组织
关注点分散                  关注点内聚
```

---

## 二、组合式函数（Composables）—— 核心概念

### 2.1 什么是 Composable

**一个以 `use` 开头、返回响应式数据和方法、可以被复用的函数。**

```js
// composables/useCounter.js
import { ref } from 'vue'

export function useCounter(initialValue = 0) {
  const count = ref(initialValue)

  const increment = () => count.value++
  const decrement = () => count.value--
  const reset = () => { count.value = initialValue }

  return { count, increment, decrement, reset }
}
```

```vue
<script setup>
import { useCounter } from './composables/useCounter'

const { count, increment, decrement, reset } = useCounter(10)
</script>

<template>
  <p>{{ count }}</p>
  <button @click="increment">+</button>
  <button @click="decrement">-</button>
  <button @click="reset">重置</button>
</template>
```

---

### 2.2 Composable 的设计原则

```
1. 命名以 use 开头       → useCounter, useFetch, useLocalStorage
2. 参数清晰              → 可配置的初始值、选项
3. 返回值明确            → 返回 { data, methods, computed }
4. 副作用可清理          → onUnmounted 中清除定时器/事件
5. 纯逻辑，不含模板      → 只处理数据和逻辑，不渲染 UI
```

---

## 三、实战 Composables 目录

### 3.1 useFetch — 数据请求

这是项目中出现频率最高的 composable。

```js
// composables/useFetch.js
import { ref, watchEffect, toValue } from 'vue'

export function useFetch(url) {
  const data = ref(null)
  const error = ref(null)
  const loading = ref(false)

  const fetchData = async () => {
    loading.value = true
    error.value = null
    data.value = null
    try {
      // toValue：无论传入 ref 还是普通字符串，都能正确取值
      const response = await fetch(toValue(url))
      if (!response.ok) throw new Error(`HTTP ${response.status}`)
      data.value = await response.json()
    } catch (e) {
      error.value = e.message
    } finally {
      loading.value = false
    }
  }

  // 监听 url 变化，自动重新请求
  watchEffect(() => {
    fetchData()
  })

  // refetch 手动重新请求
  const refetch = () => fetchData()

  return { data, error, loading, refetch }
}
```

使用：

```vue
<script setup>
import { ref } from 'vue'
import { useFetch } from './composables/useFetch'

const postId = ref(1)
const url = computed(() => `https://api.example.com/posts/${postId.value}`)

const { data: post, loading, error } = useFetch(url)
</script>

<template>
  <div v-if="loading">加载中...</div>
  <div v-else-if="error">错误：{{ error }}</div>
  <div v-else>{{ post }}</div>
  <button @click="postId++">下一篇</button>
</template>
```

---

### 3.2 useLocalStorage — 本地存储

```js
// composables/useLocalStorage.js
import { ref, watch } from 'vue'

export function useLocalStorage(key, defaultValue) {
  // 初始化：尝试从 localStorage 读取
  const storedValue = localStorage.getItem(key)
  const data = ref(storedValue ? JSON.parse(storedValue) : defaultValue)

  // 数据变化时，同步到 localStorage
  watch(data, (newVal) => {
    if (newVal === null || newVal === undefined) {
      localStorage.removeItem(key)
    } else {
      localStorage.setItem(key, JSON.stringify(newVal))
    }
  }, { deep: true })

  return data
}
```

使用：

```vue
<script setup>
import { useLocalStorage } from './composables/useLocalStorage'

// 自动持久化的主题设置
const theme = useLocalStorage('app-theme', 'light')
</script>

<template>
  <select v-model="theme">
    <option value="light">浅色</option>
    <option value="dark">深色</option>
  </select>
</template>
```

---

### 3.3 useDebounce — 防抖

```js
// composables/useDebounce.js
import { ref, watch } from 'vue'

export function useDebounce(source, delay = 300) {
  const debounced = ref(source.value)

  let timer = null
  watch(source, (val) => {
    clearTimeout(timer)
    timer = setTimeout(() => {
      debounced.value = val
    }, delay)
  })

  return debounced
}
```

```vue
<script setup>
import { ref } from 'vue'
import { useDebounce } from './composables/useDebounce'

const keyword = ref('')
const debouncedKeyword = useDebounce(keyword, 500)

// 用 debouncedKeyword 去请求，而非直接用 keyword
const { data } = useFetch(() => `api/search?q=${debouncedKeyword.value}`)
</script>
```

---

### 3.4 useMouse — 鼠标位置

```js
// composables/useMouse.js
import { ref, onMounted, onUnmounted } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  function update(event) {
    x.value = event.pageX
    y.value = event.pageY
  }

  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))

  return { x, y }
}
```

```vue
<script setup>
import { useMouse } from './composables/useMouse'

const { x, y } = useMouse()
</script>

<template>
  <p>鼠标位置：{{ x }}, {{ y }}</p>
</template>
```

---

### 3.5 useEventListener — 类型安全的通用事件监听

```js
// composables/useEventListener.js
import { onMounted, onUnmounted } from 'vue'

export function useEventListener(target, event, handler) {
  onMounted(() => target.addEventListener(event, handler))
  onUnmounted(() => target.removeEventListener(event, handler))
}
```

---

### 3.6 useMediaQuery — 响应式媒体查询

```js
// composables/useMediaQuery.js
import { ref, onMounted, onUnmounted } from 'vue'

export function useMediaQuery(query) {
  const matches = ref(false)

  let mediaQuery = null
  const update = (e) => { matches.value = e.matches }

  onMounted(() => {
    mediaQuery = window.matchMedia(query)
    matches.value = mediaQuery.matches
    mediaQuery.addEventListener('change', update)
  })

  onUnmounted(() => {
    mediaQuery?.removeEventListener('change', update)
  })

  return matches
}
```

```vue
<script setup>
const isDarkMode = useMediaQuery('(prefers-color-scheme: dark)')
const isMobile = useMediaQuery('(max-width: 768px)')
</script>
```

---

### 3.7 usePagination — 分页

```js
// composables/usePagination.js
import { ref, computed } from 'vue'

export function usePagination(items, pageSize = 10) {
  const currentPage = ref(1)
  const total = computed(() => Math.ceil(items.value.length / pageSize))

  const paginatedItems = computed(() => {
    const start = (currentPage.value - 1) * pageSize
    const end = start + pageSize
    return items.value.slice(start, end)
  })

  const goTo = (page) => {
    if (page >= 1 && page <= total.value) {
      currentPage.value = page
    }
  }
  const next = () => goTo(currentPage.value + 1)
  const prev = () => goTo(currentPage.value - 1)

  return {
    currentPage,
    total,
    paginatedItems,
    goTo,
    next,
    prev,
  }
}
```

---

## 四、Composables 之间的组合

这是最强大的能力：**composable 可以调用其他 composable**。

```js
// composables/useSearch.js
import { ref } from 'vue'
import { useDebounce } from './useDebounce'
import { useFetch } from './useFetch'

export function useSearch() {
  const keyword = ref('')
  const debouncedKeyword = useDebounce(keyword, 300)  // ← 组合 useDebounce
  const url = computed(() =>
    debouncedKeyword.value
      ? `https://api.example.com/search?q=${debouncedKeyword.value}`
      : null
  )
  const { data: results, loading } = useFetch(url)    // ← 组合 useFetch

  return { keyword, results, loading }
}
```

```
useSearch
  ├── useDebounce    ← 下层
  │    └── watch + setTimeout
  └── useFetch       ← 下层
       └── watchEffect + fetch + refs

每个 composable 只做一件事，组合起来就是复杂功能。
```

---

## 五、从 Mixins 到 Composables

### 5.1 Vue 2 Mixins 的问题

```js
// Vue 2 的 mixin
const searchMixin = {
  data() {
    return { keyword: '', results: [] }
  },
  methods: {
    search() { /* ... */ },
  },
}

// 问题一：命名冲突
const anotherMixin = {
  data() {
    return { keyword: 'xxx' }  // ← 和 searchMixin 的 keyword 冲突！
  },
  methods: {
    search() { /* ... */ }     // ← 和方法冲突！不知道用的是哪个
  }
}

// 问题二：来源不清晰
export default {
  mixins: [searchMixin, anotherMixin, thirdMixin],
  // 这个组件里 keyword 到底来自哪个 mixin？根本查不出来。
}
```

---

### 5.2 Composable 完全解决

```js
// ✅ 显式导入，来源清晰
import { useSearch } from './composables/useSearch'
import { useAnalytics } from './composables/useAnalytics'

const { keyword, results, search } = useSearch()
const { track, report } = useAnalytics()

// keyword 就是 useSearch 返回的，一清二楚
// 命名冲突？解构时直接重命名：
const { keyword: analyticsKeyword } = useAnalytics()
```

| | Mixins | Composables |
|------|------|------|
| **来源** | 不清晰（隐式混入） | 清晰（显式导入） |
| **命名冲突** | 容易冲突，不好排查 | 解构重命名，一目了然 |
| **类型推断** | 困难 | 完美（正常的 JS 函数） |
| **与其它 mixin 交互** | 隐式，不可控 | 显式调用，可控 |
| **Tree-shaking** | 不支持 | 支持 |

---

## 六、Composable 高级模式

### 6.1 返回响应式 + 非响应式

```js
export function useTimer() {
  const elapsed = ref(0)
  let intervalId = null        // 不需要是响应式的！

  const start = () => {
    intervalId = setInterval(() => {
      elapsed.value++
    }, 1000)
  }

  const stop = () => {
    clearInterval(intervalId)
  }

  // 只把需要响应式的暴露出去，intervalId 是内部状态
  return { elapsed, start, stop }
}
```

**原则**：只在模板或 computed/watch 中用到的数据才需要 `ref`/`reactive`。纯粹的计数器、标志位不需要。

---

### 6.2 带选项的 Composable

```js
export function useFetch(url, options = {}) {
  const { immediate = true, timeout = 5000, retries = 3 } = options
  // ...
}
```

```js
// 使用
const { data } = useFetch('/api/data', {
  immediate: true,
  timeout: 3000,
  retries: 2,
})
```

---

### 6.3 返回 Promise（异步 Composable）

```js
export async function useAsyncData(fetcher) {
  const data = ref(null)
  const error = ref(null)
  const loading = ref(true)

  try {
    data.value = await fetcher()
  } catch (e) {
    error.value = e
  } finally {
    loading.value = false
  }

  return { data, error, loading }
}
```

```vue
<script setup>
const { data: user } = await useAsyncData(() => fetchUser(1))
// ⚠️ 顶层 await 需要 <Suspense> 包裹父组件
</script>
```

---

### 6.4 effectScope 管理副作用

```js
import { effectScope, ref, watch, onUnmounted } from 'vue'

export function useFeature() {
  const scope = effectScope()  // 创建独立的作用域

  const data = ref(null)

  scope.run(() => {
    // 这里面所有的 watch/watchEffect/computed 都被 scope 跟踪
    watch(data, () => {
      console.log('data changed')
    })
  })

  // 组件卸载时，一次性停止所有副作用
  onUnmounted(() => scope.stop())

  return { data }
}
```

为什么用 `effectScope`？普通的 `watch` 创建后会一直存在直到组件卸载。但如果 composable 被多次调用但没有正确清理，副作用的引用会堆积。`effectScope` 提供了**一次性清理所有副作用**的能力，适合复杂的 composable。

---

### 6.5 状态共享（单例模式）

普通的 composable 在每个组件中创建独立状态。如果需要**跨组件共享状态**：

```js
// composables/useGlobalCounter.js
import { ref } from 'vue'

// ⚠️ 状态定义在函数外部 → 模块级单例
const count = ref(0)

export function useGlobalCounter() {
  const increment = () => count.value++
  const decrement = () => count.value--
  return { count, increment, decrement }
}
```

```vue
<!-- 组件 A -->
<script setup>
const { count, increment } = useGlobalCounter()
count.value // → 0
increment() // count 变成 1
</script>

<!-- 组件 B -->
<script setup>
const { count } = useGlobalCounter()
count.value // → 1  ← 和组件 A 共享同一个状态！
</script>
```

> ⚠️ **注意**：这是**模块级共享**，不同于 Pinia。适合简单场景（如全局 Toast、Loading），复杂场景还是用 Pinia。

---

## 七、Composable 命名规范

```js
// 社区公认的命名约定

useFetch()          // 数据获取
useDebounce()       // 防抖
useThrottle()       // 节流
useLocalStorage()   // 本地存储
useMouse()          // 鼠标
useEventListener()  // 事件监听
useMediaQuery()     // 媒体查询
useForm()           // 表单
usePagination()     // 分页
useInfiniteScroll() // 无限滚动
useVirtualList()    // 虚拟列表
useWebSocket()      // WebSocket
useAuth()           // 认证
useTheme()          // 主题
useDarkMode()       // 暗黑模式

// 模式：use<Feature>
// 清晰表达：这是一个 Vue Composable
```

---

## 八、VueUse — 社区标杆

[VueUse](https://vueuse.org/) 是 Vue 生态最优秀的 Composable 库，包含 200+ 个高质量 Composable。

```bash
npm install @vueuse/core
```

```js
import { useDark, useToggle } from '@vueuse/core'

const isDark = useDark()
const toggleDark = useToggle(isDark)
// 三行代码搞定暗黑模式，浏览器偏好自动跟随，加入 localStorage 持久化
```

**学习建议**：读 VueUse 源码是学习 Composable 最佳实践的最佳方式。代码短小精悍，每个函数职责单一，命名清晰。

---

## 九、实战：从零构建一个完整功能

### 需求：可搜索的用户列表（带分页、加载状态、错误处理）

```vue
<script setup>
import { ref, computed } from 'vue'
import { useFetch } from './composables/useFetch'
import { useDebounce } from './composables/useDebounce'
import { usePagination } from './composables/usePagination'

// 1. 搜索
const keyword = ref('')
const debouncedKeyword = useDebounce(keyword, 300)

// 2. 数据获取
const url = computed(() =>
  `https://api.example.com/users?q=${debouncedKeyword.value}`
)
const { data: users, loading, error } = useFetch(url)

// 3. 分页
const { paginatedItems: displayedUsers, currentPage, total, next, prev } =
  usePagination(computed(() => users.value ?? []), 10)

// 30 行代码，功能完整，逻辑清晰
</script>

<template>
  <input v-model="keyword" placeholder="搜索用户..." />

  <div v-if="loading">加载中...</div>
  <div v-else-if="error">{{ error }}</div>
  <ul v-else>
    <li v-for="user in displayedUsers" :key="user.id">
      {{ user.name }}
    </li>
  </ul>

  <div>
    <button :disabled="currentPage <= 1" @click="prev">上一页</button>
    <span>{{ currentPage }} / {{ total }}</span>
    <button :disabled="currentPage >= total" @click="next">下一页</button>
  </div>
</template>
```

---

## 十、面试题

### 1. Composition API 和 Options API 有什么区别？

> Options API 按 `data`/`methods`/`watch` 等选项组织代码，同一功能的相关逻辑被分散到不同选项中。Composition API 按功能关注点组织代码，将同一功能的数据、方法、侦听放在一起。
>
> Composition API 的优势：更好的逻辑复用（Composables）、更好的 TypeScript 支持、更清晰的代码组织、支持 tree-shaking。

---

### 2. Composables 和 React Hooks 的区别？

> 最大的区别：
>
> 1. **调用时机**：Composables 在 `setup()` 中**只执行一次**，React Hooks 在**每次渲染都执行**。
> 2. **心智模型**：Vue 响应式系统自动追踪依赖，不需要手动声明依赖数组（`useEffect(fn, [dep])`）。
> 3. **性能**：Composables 只在 setup 时运行，React Hooks 每次渲染都跑一遍，需要 `useMemo`/`useCallback` 手动优化。

---

### 3. 怎么把一个 Options API 组件迁移到 Composition API？

> 1. `data()` → `ref()` / `reactive()`
> 2. `computed` → `computed()`
> 3. `watch` → `watch()` / `watchEffect()`
> 4. `methods` → 普通函数
> 5. 生命周期钩子 → `onMounted()` 等
> 6. `this.xxx` → 变量直接引用
> 7. 可复用的逻辑抽成 Composable

---

### 4. Composables 怎么处理同一个实例的状态共享？

> 将状态定义在 Composable 函数外部（模块作用域），即可实现调用该 Composable 的所有组件共享同一份状态。适用于轻量场景（全局通知、Loading）。但复杂跨组件状态管理仍推荐 Pinia。

---

## 十一、总结

```
Composition API 思维转变：
  从「按选项组织」→「按功能组织」

Composable 设计：
  use 开头 → 接收参数 → 返回响应式数据和方法 → 内部管理副作用

Composable 的价值：
  1. 逻辑复用    → 跨组件共享功能
  2. 代码组织    → 相关代码聚在一起
  3. 可测试      → 纯函数，易于单元测试
  4. 可组合      → Composable 可以互相嵌套

关键认知：
  Composable 本质就是普通的 JavaScript 函数
  只是它们利用了 Vue 的响应式系统
```

下一篇预告：**内置组件** — KeepAlive / Teleport / Suspense / Transition / TransitionGroup。
