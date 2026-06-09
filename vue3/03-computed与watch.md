# Vue 3 computed 与 watch

> computed 是响应式的「派生」，watch 是响应式的「副作用」。一文搞懂两者的原理、区别和使用场景。

---

## 一、computed — 计算属性

### 1.1 基本用法

```vue
<script setup>
import { ref, computed } from 'vue'

const firstName = ref('三')
const lastName = ref('张')

// 基本写法
const fullName = computed(() => {
  return lastName.value + firstName.value
})

// 可读写写法
const fullName2 = computed({
  get() {
    return lastName.value + firstName.value
  },
  set(val) {
    lastName.value = val[0]
    firstName.value = val.slice(1)
  }
})
</script>

<template>
  <p>{{ fullName }}</p>  <!-- "张三" -->
</template>
```

---

### 1.2 computed 四大特性

| 特性 | 说明 |
|------|------|
| **缓存** | 依赖不变，重复访问返回缓存值，不重新计算 |
| **惰性** | 定义时不执行，第一次访问时才计算 |
| **依赖追踪** | 自动追踪内部用到的响应式数据 |
| **只读** | 默认返回值是只读的 ref（除非定义 setter） |

---

### 1.3 缓存 vs 不缓存的对比

```vue
<script setup>
import { ref, computed } from 'vue'

const count = ref(0)

// ✅ computed：有缓存
const computedDouble = computed(() => {
  console.log('computed 计算了')
  return count.value * 2
})

// ❌ 方法：每次渲染都重新执行
function methodDouble() {
  console.log('方法执行了')
  return count.value * 2
}
</script>

<template>
  <!-- 访问 3 次 computed，只计算 1 次 -->
  <p>{{ computedDouble }}</p>
  <p>{{ computedDouble }}</p>
  <p>{{ computedDouble }}</p>

  <!-- 访问 3 次方法，执行 3 次 -->
  <p>{{ methodDouble() }}</p>
  <p>{{ methodDouble() }}</p>
  <p>{{ methodDouble() }}</p>
</template>
```

**结论**：模板中多次使用同一个计算属性时，computed 只算一次，方法每次都算。

---

### 1.4 什么时候用 computed vs 方法

```
用 computed：
  → 值依赖其他响应式数据
  → 会被多次访问
  → 计算成本较高

用方法：
  → 需要传参
  → 不需要缓存
  → 只是简单格式化
```

---

### 1.5 computed 的原理

核心是 **懒执行 + 脏检查**：

```js
// 简化版 computed 实现
class ComputedRefImpl {
  constructor(getter) {
    this._getter = getter
    this._dirty = true       // 🏁 脏标记，初始为 true
    this._value = undefined  // 缓存值
    this._effect = new ReactiveEffect(getter, () => {
      // 当依赖变化时，只把 dirty 设为 true
      // 不立即重新计算！
      if (!this._dirty) {
        this._dirty = true
        trigger(this, 'value')
      }
    })
  }

  get value() {
    if (this._dirty) {       // 脏了才重新计算
      this._value = this._effect.run()
      this._dirty = false    // 计算完，标记干净
    }
    track(this, 'value')     // 依赖收集
    return this._value       // 返回缓存值
  }
}
```

**流程总结**：

```
第一次访问 computed.value
  └─ dirty = true → 执行 getter → dirty = false → 返回缓存

依赖数据改变
  └─ 触发 effect 回调 → dirty = true → 通知 computed 的订阅者

再次访问 computed.value（依赖没变）
  └─ dirty = false → 直接返回缓存，不计算
```

---

### 1.6 computed 接受一个 ref 数组（Vue 3.4+）

```js
const a = ref(1)
const b = ref(2)

// Vue 3.4+：显式声明依赖
const sum = computed(() => a.value + b.value)
// 等价写法——显式声明依赖以优化
const sum2 = computed([a, b], ([aVal, bVal]) => aVal + bVal)
```

---

## 二、watch — 监听器

### 2.1 基本用法

```vue
<script setup>
import { ref, watch } from 'vue'

const count = ref(0)

// 监听单个 ref
watch(count, (newVal, oldVal) => {
  console.log(`count 从 ${oldVal} 变为 ${newVal}`)
})
</script>
```

---

### 2.2 四种监听数据源

```js
// ① ref
const count = ref(0)
watch(count, (newVal, oldVal) => { /* ... */ })

// ② reactive 对象的属性（getter 函数）
const state = reactive({ count: 0 })
watch(
  () => state.count,
  (newVal, oldVal) => { /* ... */ }
)

// ③ 多个数据源（数组）
watch(
  [count, () => state.name],
  ([newCount, newName], [oldCount, oldName]) => { /* ... */ }
)

// ④ reactive 整个对象
watch(state, (newVal, oldVal) => { /* ... */ })
// ⚠️ 这种情况会自动开启 deep，新旧值引用相同
```

---

### 2.3 三个常用选项

```js
watch(source, callback, {
  immediate: true,   // 立即执行一次 callback
  deep: true,        // 深度监听
  flush: 'post',     // 'pre'(默认) | 'post' | 'sync'
})
```

---

### 2.4 deep — 深度监听

```js
const state = reactive({
  user: {
    name: 'Alice',
    profile: { age: 25 },
  },
})

// ❌ 默认不深度监听（监听整个 reactive 对象除外）
watch(
  () => state.user,
  (newVal) => {
    console.log('不会触发：直接修改嵌套属性')
  }
)

state.user.profile.age = 26  // ❌ 不触发

// ✅ deep: true 深度监听
watch(
  () => state.user,
  (newVal) => {
    console.log('触发了')
  },
  { deep: true }
)

state.user.profile.age = 26  // ✅ 触发

// ⚠️ 直接 watch 一个 reactive 对象默认就是 deep
watch(state, () => {
  console.log('任何属性变化都会触发')
})
```

**`deep` 的性能代价**：

> 开启 deep 后，Vue 会递归遍历所有嵌套属性进行依赖收集。对大对象开启 deep 可能造成性能问题。

---

### 2.5 immediate — 立即执行

```js
const count = ref(0)

// 默认：count 变化时才执行
watch(count, (newVal, oldVal) => {
  console.log(oldVal, '→', newVal)
})

// immediate: true：立即执行一次
watch(count, (newVal, oldVal) => {
  console.log(oldVal, '→', newVal)
  // 首次执行时 oldVal 是 undefined
}, { immediate: true })
// 立即输出：undefined → 0
```

**典型场景**：页面初始化后从 API 获取数据。

```js
const userId = ref(1)

watch(userId, async (newId) => {
  const data = await fetchUser(newId)
  user.value = data
}, { immediate: true })  // 初始就根据 userId 拉数据
```

---

### 2.6 flush — 回调时机

```vue
<script setup>
import { ref, watch } from 'vue'

const count = ref(0)

// flush: 'pre'（默认）—— 组件更新前
watch(count, () => {
  console.log('pre:', document.getElementById('count')?.textContent)
  // 此时 DOM 还没更新！
})

// flush: 'post' —— 组件更新后
watch(count, () => {
  console.log('post:', document.getElementById('count')?.textContent)
  // 此时 DOM 已经更新
  //（watchEffect 默认就是 post）
}, { flush: 'post' })

// flush: 'sync' —— 同步触发（极少用，有性能风险）
watch(count, () => { /* 数据变就同步触发 */ }, { flush: 'sync' })
</script>

<template>
  <p id="count">{{ count }}</p>
</template>
```

---

### 2.7 watch 返回的 stop 函数

```js
const stop = watch(count, (newVal) => {
  if (newVal > 10) {
    stop()  // 停止监听
  }
})
```

---

## 三、watchEffect — 自动追踪的副作用

### 3.1 基本用法

```js
import { ref, watchEffect } from 'vue'

const count = ref(0)
const name = ref('Alice')

// watchEffect 自动追踪内部访问的所有响应式数据
watchEffect(() => {
  console.log('count:', count.value, 'name:', name.value)
})

count.value++  // 触发
name.value = 'Bob'  // 也触发
```

**不需要手动声明依赖**，函数体内访问了什么就监听什么。

---

### 3.2 watchEffect vs watch

| | watch | watchEffect |
|------|------|------|
| 依赖声明 | 手动指定数据源 | 自动追踪 |
| 初始执行 | `immediate: true` 才执行 | **默认立即执行** |
| 访问旧值 | ✅ `oldVal` | ❌ 无法获取 |
| 精确控制 | ✅ 可以 watch 特定属性 | ❌ 访问什么监听什么 |
| 使用场景 | 需要比较新旧值、精确控制 | 依赖自动收集、不需要旧值 |

---

### 3.3 使用场景

```js
// ✅ watchEffect 适用：自动收集依赖
const todoId = ref(1)
const data = ref(null)

watchEffect(async () => {
  // 自动追踪 todoId.value 和 data.value
  data.value = await fetchTodo(todoId.value)
  console.log('数据更新完成')
})

// ✅ watch 适用：需要比较新旧值
watch(todoId, async (newId, oldId) => {
  console.log(`从 ${oldId} 切换到 ${newId}`)
  data.value = await fetchTodo(newId)
})
```

---

### 3.4 watchEffect 的副作用清理

```js
watchEffect((onCleanup) => {
  const controller = new AbortController()

  // 注册清理函数
  onCleanup(() => {
    controller.abort()  // 取消上一次未完成的请求
  })

  fetch(url, { signal: controller.signal })
    .then(res => res.json())
    .then(data => { result.value = data })
})
```

---

### 3.5 watchEffect 的实现原理

```js
function watchEffect(effectFn) {
  const runner = new ReactiveEffect(effectFn)

  // 首次立即执行
  runner.run()

  // 返回 stop 函数
  return () => runner.stop()
}

class ReactiveEffect {
  constructor(fn, scheduler) {
    this.fn = fn
    this.scheduler = scheduler
    this.deps = []  // 当前 effect 被哪些属性依赖
  }

  run() {
    activeEffect = this  // 标记为当前正在执行的 effect
    return this.fn()     // 执行 → 触发 getter → track 收集依赖
  }
}
```

每次 `run()` 时重新执行函数，触发新的 `track` 收集，旧的依赖在下次执行前被清理。

---

## 四、watchPostEffect / watchSyncEffect / watchPreEffect

```js
import { watchPostEffect, watchSyncEffect, watchPreEffect } from 'vue'

// 这三个是 watchEffect + flush 的语法糖：

// watchEffect(() => {...}, { flush: 'post' })  等价于
watchPostEffect(() => { /* DOM 更新后执行 */ })

// watchEffect(() => {...}, { flush: 'sync' })   等价于
watchSyncEffect(() => { /* 同步执行 */ })

// watchEffect(() => {...}, { flush: 'pre' })    等价于（这是 watchEffect 的默认行为？非也，watchEffect 默认是 pre）
watchPreEffect(() => { /* 组件更新前执行 */ })
```

---

## 五、computed 与 watch 的关系

### 5.1 什么时候用哪个

```
              ┌──────────────────┐
              │ 需要返回值吗？    │
              └──────┬───────────┘
                Yes  │  No
              ┌──────▼───────┐
              │   computed   │
              └──────────────┘
                     │
         ┌───────────┼───────────┐
         │ 不需要返回值，但要副作用 │
         └───────────┬───────────┘
                     │
              ┌──────▼───────────┐
              │ 需要新旧值比较？  │
              └──────┬───────────┘
                Yes  │  No
              ┌──────▼───────┐
              │    watch     │
              └──────────────┘
                     │
              ┌──────▼───────┐
              │ watchEffect  │
              └──────────────┘
```

---

### 5.2 三者对比速查

| | computed | watch | watchEffect |
|------|:---:|:---:|:---:|
| 有返回值 | ✅ | ❌ | ❌ |
| 缓存 | ✅ | ❌ | ❌ |
| 初次执行 | 第一次访问时 | `immediate` 控制 | ✅ 立即执行 |
| 访问旧值 | ❌ | ✅ | ❌ |
| 手动声明依赖 | 自动 | ✅ 手动 | 自动 |
| 主要用途 | 派生数据 | 执行副作用 | 执行副作用 |
| 可读写 | ✅ 设置 setter | ❌ | ❌ |

---

## 六、实战场景与最佳实践

### 6.1 搜索防抖

```vue
<script setup>
import { ref, watch } from 'vue'

const keyword = ref('')
const results = ref([])

// 用户停止输入 300ms 后才请求
let timer = null
watch(keyword, (val) => {
  clearTimeout(timer)
  timer = setTimeout(async () => {
    results.value = await searchAPI(val)
  }, 300)
})
</script>
```

**更好的方式**（用 composable）：

```js
import { ref, watch } from 'vue'

export function useDebouncedWatch(source, callback, delay = 300) {
  let timer = null
  watch(source, (val) => {
    clearTimeout(timer)
    timer = setTimeout(() => callback(val), delay)
  })
}
```

---

### 6.2 表单校验（computed）

```vue
<script setup>
import { ref, computed } from 'vue'

const password = ref('')
const confirmPassword = ref('')

const isPasswordValid = computed(() => password.value.length >= 6)
const isPasswordMatch = computed(() => password.value === confirmPassword.value)
const canSubmit = computed(() => isPasswordValid.value && isPasswordMatch.value)

const errorMessage = computed(() => {
  if (!password.value) return ''
  if (!isPasswordValid.value) return '密码至少 6 位'
  if (confirmPassword.value && !isPasswordMatch.value) return '两次密码不一致'
  return ''
})
</script>
```

---

### 6.3 路由参数监听

```vue
<script setup>
import { watch } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()

watch(
  () => route.params.id,
  async (newId) => {
    // 路由参数变化 → 重新加载数据
    const data = await fetchPost(newId)
    post.value = data
  },
  { immediate: true }
)
</script>
```

---

### 6.4 列表过滤与排序（computed）

```vue
<script setup>
import { ref, computed } from 'vue'

const items = ref([...])           // 原始数据
const filterType = ref('all')      // 过滤条件
const sortOrder = ref('asc')       // 排序

const filteredItems = computed(() => {
  let result = items.value
  if (filterType.value !== 'all') {
    result = result.filter(item => item.type === filterType.value)
  }
  return result.sort((a, b) => {
    return sortOrder.value === 'asc'
      ? a.price - b.price
      : b.price - a.price
  })
})
</script>
```

---

### 6.5 避免在 computed 中产生副作用

```js
// ❌ 不要在 computed 中做这些事：
const bad = computed(() => {
  fetch('/api/data')          // 副作用：网络请求
  document.title = 'xxx'      // 副作用：修改 DOM
  count.value++               // 副作用：修改其他状态
  return something
})

// ✅ computed 只做纯计算
const good = computed(() => {
  return items.value.filter(i => i.active)  // 纯计算
})

// ✅ 副作用用 watch
watch(good, (filtered) => {
  document.title = `共 ${filtered.length} 条`
})
```

---

## 七、effectScope — 管理副作用生命周期

### 7.1 使用场景

```js
import { effectScope, ref, watch, computed } from 'vue'

// 创建一个作用域
const scope = effectScope()

scope.run(() => {
  // 这里面创建的 watch、watchEffect、computed 都会被跟踪
  const doubled = computed(() => count.value * 2)

  watch(doubled, (val) => {
    console.log(val)
  })

  watchEffect(() => {
    console.log(count.value)
  })
})

// 一键停止作用域内的所有副作用
scope.stop()  // 所有 watch / watchEffect / computed 全部停掉
```

**典型场景**：Composable 中统一管理副作用：

```js
// useFeature.js
import { effectScope } from 'vue'

export function useFeature() {
  const scope = effectScope()

  scope.run(() => {
    // ... 多个 watch / watchEffect
  })

  onUnmounted(() => scope.stop())  // 组件卸载时清理

  return { /* 暴露的 API */ }
}
```

---

## 八、面试题

### 8.1 computed 和 watch 的区别

> **computed**：
> - 有返回值（派生一个新的值）
> - 有缓存，依赖不变不重新计算
> - 惰性求值，第一次访问才执行
> - 不应产生副作用
>
> **watch**：
> - 无返回值（执行副作用）
> - 无缓存
> - 可选择 `immediate` 立即执行
> - 可获取新旧值
> - 用于异步操作、DOM 操作等副作用

---

### 8.2 computed 怎么实现缓存

> 通过 `_dirty` 标记位实现：
> - 首次访问时 `_dirty = true`，执行 getter，结果缓存，`_dirty = false`
> - 依赖变化时只将 `_dirty` 置为 `true`，不立即计算
> - 下次访问时发现 `_dirty = true`，重新计算并缓存
> - 如果 `_dirty = false` 且依赖没变，直接返回缓存值

---

### 8.3 watchEffect 和 watch 的区别

> - **watchEffect** 自动追踪依赖，立即执行；**watch** 手动声明数据源，惰性执行
> - **watchEffect** 无法获取旧值；**watch** 可以
> - **watchEffect** 默认在组件更新前执行；**watch** 默认也在更新前，但更灵活

---

### 8.4 为什么 computed 不能做异步

> computed 依赖同步返回值的 getter 函数。异步操作（如 `async/await`）返回 Promise，computed 拿到的是 Promise 对象而非最终值，无法实现缓存和依赖追踪。异步场景应使用 `watch` + 赋值。

---

## 九、总结

```
computed     → 派生一个新的响应式值，自动缓存，纯函数
watch        → 监听变化后执行副作用，可比较新旧值
watchEffect  → 自动追踪依赖的副作用，立即执行
effectScope  → 统一管理副作用生命周期

关键原则：
  computed 里不要有副作用
  watch 里不要修改监听的数据（避免死循环）
  用对工具，代码简洁且性能好
```

下一篇文章预告：**组件基础与通信**——props、emits、provide/inject、v-model。
