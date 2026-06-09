# TypeScript 与 Vue 3

> Vue 3 从设计之初就为 TypeScript 而生。本文覆盖在 Vue 3 中写 TypeScript 的完整实践。

---

## 一、为什么 Vue 3 + TypeScript 是绝配

| Vue 2 + TS | Vue 3 + TS |
|------|------|
| 类型推断困难 | `ref` / `reactive` 自动推断 |
| 需要 `vue-class-component` | 原生 `<script setup lang="ts">` 支持 |
| Props 类型定义繁琐 | `defineProps<T>()` 一行搞定 |
| this 类型混乱 | 无 `this`，函数作用域天然类型安全 |

---

## 二、项目初始化

```bash
# Vite 创建（推荐）
npm create vue@latest

# 勾选 TypeScript 选项即可
# 或手动创建
npm create vite@latest my-app -- --template vue-ts
```

---

## 三、组件 Props 类型

### 3.1 defineProps 的四种写法

```vue
<script setup lang="ts">
// ① 运行时声明（常用简单场景）
const props = defineProps({
  name: { type: String, required: true },
  age: { type: Number, default: 18 },
})

// ② 纯类型声明（推荐——最简洁）
const props = defineProps<{
  name: string
  age?: number
  tags: string[]
  user: { id: number; name: string }
}>()

// ③ 类型声明 + 默认值
const props = withDefaults(defineProps<{
  name?: string
  age?: number
  disabled?: boolean
}>(), {
  name: '匿名用户',
  age: 18,
  disabled: false,
})

// ④ 从 interface / type 引用
interface User {
  id: number
  name: string
  role: 'admin' | 'editor' | 'viewer'
}
const props = defineProps<{
  user: User
  onUpdate: (user: User) => void
}>()
</script>
```

---

### 3.2 Props 的只读性

```ts
const props = defineProps<{ count: number }>()

props.count++  // ❌ TS 报错：props 是只读的
// 正确的做法：复制或触发 emit
```

---

## 四、Emits 类型

```vue
<script setup lang="ts">
// ① 运行时声明
const emit = defineEmits(['update', 'delete'])

// ② 类型声明（推荐）
const emit = defineEmits<{
  // (event: '事件名', 参数1: 类型, 参数2: 类型): void
  update: [id: number, name: string]
  delete: [id: number]
  submit: [data: { email: string; password: string }]
}>()

// 使用：类型安全
emit('update', 1, 'Alice')      // ✅
emit('update', 'wrong', 2)      // ❌ TS 报错：参数类型不对
emit('delete', 1)               // ✅
emit('delete')                  // ❌ TS 报错：缺少参数
</script>
```

---

### 4.1 Vue 3.3+ 语法

```vue
<script setup lang="ts">
// 更简洁的写法（Vue 3.3+）
const emit = defineEmits<{
  'update:modelValue': [value: string]
  change: [id: number]
}>()
</script>
```

---

## 五、ref 与 reactive 类型

### 5.1 ref 的类型推断

```ts
import { ref } from 'vue'

// ✅ 自动推断
const count = ref(0)           // Ref<number>
const name = ref('Alice')      // Ref<string>
const isActive = ref(true)     // Ref<boolean>

// ✅ 显式声明类型
const user = ref<User | null>(null)
const list = ref<string[]>([])

// ✅ 复杂类型
const data = ref<{
  items: { id: number; title: string }[]
  total: number
}>({
  items: [],
  total: 0,
})

// ⚠️ 注意：ref() 的参数类型和 Ref 的泛型是「协变」的
const a: Ref<number> = ref(1)
const b: Ref<string | number> = a  // ✅
```

---

### 5.2 reactive 的类型

```ts
import { reactive } from 'vue'

// ✅ 自动推断
const state = reactive({
  count: 0,
  user: { name: 'Alice' } as User,
})

// ✅ 显式声明
interface AppState {
  count: number
  loading: boolean
  user: User | null
}
const state = reactive<AppState>({
  count: 0,
  loading: false,
  user: null,
})

// ⚠️ reactive 不支持基本类型
const n = reactive(0)  // ❌ TS 报错
```

---

### 5.3 解包类型

```ts
import { ref, computed, Ref, ComputedRef } from 'vue'

const count = ref(0)
// count 的类型是 Ref<number>
// count.value 的类型是 number

const doubled = computed(() => count.value * 2)
// doubled 的类型是 ComputedRef<number>

// 提取 Ref 内部类型
type CountType = Ref<number>['value']  // number
```

---

## 六、Composable 类型

### 6.1 基本类型签名

```ts
// composables/useFetch.ts
import { ref, type Ref } from 'vue'

interface UseFetchReturn<T> {
  data: Ref<T | null>
  error: Ref<string | null>
  loading: Ref<boolean>
  refetch: () => Promise<void>
}

export function useFetch<T>(url: string | Ref<string>): UseFetchReturn<T> {
  const data = ref<T | null>(null) as Ref<T | null>
  const error = ref<string | null>(null)
  const loading = ref(false)

  // ...

  return { data, error, loading, refetch }
}
```

```vue
<script setup lang="ts">
interface Post {
  id: number
  title: string
  content: string
}

// 使用时：自动推断 T = Post
const { data, loading, error } = useFetch<Post>('/api/posts/1')
// data 的类型：Ref<Post | null>
</script>
```

---

### 6.2 泛型 Composable

```ts
// composables/useList.ts
export function useList<T>(initialItems: T[] = []) {
  const items = ref<T[]>(initialItems) as Ref<T[]>
  const selected = ref<T | null>(null) as Ref<T | null>

  function add(item: T) {
    items.value.push(item)
  }

  function remove(predicate: (item: T) => boolean) {
    items.value = items.value.filter((item) => !predicate(item))
  }

  function select(item: T) {
    selected.value = item
  }

  return { items, selected, add, remove, select }
}
```

```vue
<script setup lang="ts">
interface Todo {
  id: number
  text: string
  done: boolean
}

const { items, add, remove } = useList<Todo>()

add({ id: 1, text: '学习 TS', done: false })  // ✅ 类型检查通过
add({ id: 2, text: '学习 Vue' })              // ❌ 缺少 done
</script>
```

---

### 6.3 接受 Ref 或普通值

```ts
import { type Ref, type MaybeRef, unref } from 'vue'

// MaybeRef<T> = T | Ref<T>  （Vue 3.3+ 内置）

export function useTitle(title: MaybeRef<string>) {
  // unref 自动取出值
  document.title = unref(title)
}

// 使用
useTitle('首页')             // ✅ 传字符串
useTitle(ref('关于我们'))    // ✅ 传 ref
useTitle(computed(() => ...)) // ✅ 传 computed
```

---

### 6.4 返回类型不宽的技巧

```ts
// ❌ 类型被拓宽
const count = ref(0)   // Ref<number>，丢失了「0」这个精确类型

// ✅ 用 as const
const status = ref('idle' as const)  // Ref<'idle'>
const options = ref(['a', 'b'] as const)  // Ref<readonly ['a', 'b']>

// ✅ 用泛型
const status = ref<'idle' | 'loading' | 'success' | 'error'>('idle')
```

---

## 七、模板引用（Template Refs）

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

// ① DOM 元素引用
const inputRef = ref<HTMLInputElement | null>(null)

onMounted(() => {
  inputRef.value?.focus()  // ✅ 有完整的 HTMLInputElement 方法
})

// ② 组件实例引用
import MyModal from './MyModal.vue'

// 需要从组件中导出类型
const modalRef = ref<InstanceType<typeof MyModal> | null>(null)

function openModal() {
  modalRef.value?.open()  // ✅ 类型安全的访问暴露方法
}
</script>

<template>
  <input ref="inputRef" />
  <MyModal ref="modalRef" />
</template>
```

**更好的做法**：组件暴露的类型单独导出

```vue
<!-- MyModal.vue -->
<script setup lang="ts">
export interface ModalExpose {
  open: () => void
  close: () => void
}

const visible = ref(false)
const open = () => { visible.value = true }
const close = () => { visible.value = false }

defineExpose<ModalExpose>({ open, close })
</script>
```

```vue
<!-- 使用方 -->
<script setup lang="ts">
import MyModal from './MyModal.vue'
import type { ModalExpose } from './MyModal.vue'

const modalRef = ref<ModalExpose | null>(null)
modalRef.value?.open()  // ✅
</script>
```

---

## 八、事件处理类型

```vue
<script setup lang="ts">
// DOM 事件
function handleClick(event: MouseEvent) {
  console.log(event.clientX, event.clientY)
}

function handleInput(event: Event) {
  const target = event.target as HTMLInputElement
  console.log(target.value)
}

function handleKeydown(event: KeyboardEvent) {
  if (event.key === 'Enter') {
    // ...
  }
}

function handleSubmit(event: Event) {
  event.preventDefault()
  // ...
}
</script>

<template>
  <button @click="handleClick">点击</button>
  <input @input="handleInput" @keydown="handleKeydown" />
  <form @submit="handleSubmit">...</form>
</template>
```

---

## 九、Provide / Inject 类型

```ts
// types.ts —— 集中管理 InjectionKey
import type { InjectionKey, Ref } from 'vue'

export interface User {
  id: number
  name: string
  role: string
}

export const USER_KEY: InjectionKey<Ref<User | null>> = Symbol('user')
export const THEME_KEY: InjectionKey<Ref<'light' | 'dark'>> = Symbol('theme')
export const UPDATE_USER_KEY: InjectionKey<(user: User) => void> = Symbol('updateUser')
```

```vue
<!-- 祖先组件 -->
<script setup lang="ts">
import { provide, ref } from 'vue'
import { USER_KEY, UPDATE_USER_KEY, type User } from './types'

const user = ref<User | null>(null)

provide(USER_KEY, user)
provide(UPDATE_USER_KEY, (newUser: User) => {
  user.value = newUser
})
</script>

<!-- 后代组件 -->
<script setup lang="ts">
import { inject } from 'vue'
import { USER_KEY, UPDATE_USER_KEY } from './types'

const user = inject(USER_KEY)   // 类型自动推断为 Ref<User | null> | undefined
const updateUser = inject(UPDATE_USER_KEY)  // 类型自动推断

// 用默认值消除 undefined
const userSafe = inject(USER_KEY, ref(null))
</script>
```

---

## 十、defineModel 类型（Vue 3.4+）

```vue
<!-- 子组件 -->
<script setup lang="ts">
// 基本
const model = defineModel()               // Ref<string>（如果父组件传 string）

// 带类型
const count = defineModel<number>()        // Ref<number | undefined>

// 必填
const name = defineModel<string>({ required: true })

// 带默认值
const age = defineModel<number>({ default: 0 })

// 多个 model
const firstName = defineModel<string>('firstName')
const lastName = defineModel<string>('lastName')

// 带修饰符
const [model, modifiers] = defineModel<string>()
// modifiers 类型：{ capitalize?: boolean; uppercase?: boolean; ... }
</script>

<template>
  <input v-model="model" />
</template>
```

---

## 十一、defineSlots 类型（Vue 3.3+）

```vue
<script setup lang="ts">
defineSlots<{
  default?: (props: { msg: string }) => any
  header?: (props: { title: string }) => any
  footer?: (props: Record<string, never>) => any
}>()
</script>

<template>
  <div>
    <slot name="header" title="页面标题" />
    <slot msg="hello" />
    <slot name="footer" />
  </div>
</template>
```

使用时，父组件获得插槽的类型提示：

```vue
<MyComponent>
  <template #header="{ title }">
    <!-- title 类型自动推断为 string -->
    <h1>{{ title }}</h1>
  </template>

  <template #default="{ msg }">
    <!-- msg 类型自动推断为 string -->
    <p>{{ msg }}</p>
  </template>
</MyComponent>
```

---

## 十二、全局类型声明

```ts
// env.d.ts 或 global.d.ts

// 声明 .vue 文件模块
declare module '*.vue' {
  import type { DefineComponent } from 'vue'
  const component: DefineComponent<{}, {}, any>
  export default component
}

// 扩展全局组件类型
declare module 'vue' {
  export interface GlobalComponents {
    MyButton: typeof import('./components/MyButton.vue')['default']
  }
}

// 扩展 ComponentCustomProperties
declare module 'vue' {
  interface ComponentCustomProperties {
    $formatDate: (date: Date) => string
  }
}
```

---

## 十三、tsconfig 推荐配置

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "jsx": "preserve",
    "noEmit": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "paths": {
      "@/*": ["./src/*"]
    },
    "types": ["vite/client"]
  },
  "include": ["src/**/*.ts", "src/**/*.tsx", "src/**/*.vue", "env.d.ts"]
}
```

**关键点**：
- `moduleResolution: "bundler"` — Vue 3.3+ 推荐，支持 import from 的条件导出
- `strict: true` — 开启所有严格检查
- `isolatedModules: true` — Vite 要求
- `types: ["vite/client"]` — 引入 Vite 的类型定义（`import.meta.env` 等）

---

## 十四、常见 TS 错误和解决

### 14.1 ref 可能为 null

```ts
const user = ref<User | null>(null)

// ❌ TS 报错：user.value 可能为 null
console.log(user.value.name)

// ✅ 方案 1：可选链
console.log(user.value?.name)

// ✅ 方案 2：类型守卫
if (user.value) {
  console.log(user.value.name)  // ✅ TS 知道非 null
}

// ✅ 方案 3：非空断言（确定非 null 时）
console.log(user.value!.name)
```

---

### 14.2 模板中不需要 .value 导致的困惑

```ts
// TS 中访问 count.value 正常
count.value++  // ✅

// 但类型系统里 count 是 Ref<number>，不是 number
function useCalc(n: number) { /* ... */ }
// useCalc(count)       // ❌ Ref<number> 不等于 number
useCalc(count.value)    // ✅

// 工具：MaybeRef
import type { MaybeRef, unref } from 'vue'
function useCalc(n: MaybeRef<number>) {
  const val = unref(n)  // ✅ 自动取出 value
}
```

---

### 14.3 组件的类型导出

```vue
<!-- ❌ 常见错误：ts 编译器看不懂 .vue 文件 -->
<script setup lang="ts">
export interface Props {
  name: string
}
</script>

<!-- ✅ 方案：使用 vue-component-type-helpers -->
<!-- 安装：npm i -D vue-component-type-helpers -->
```

```ts
// 使用方
import type { ComponentProps } from 'vue-component-type-helpers'
import MyComponent from './MyComponent.vue'

type MyProps = ComponentProps<typeof MyComponent>
// { name: string }
```

---

## 十五、面试题

### 1. defineProps 的类型声明有哪几种方式？

> 1. 运行时声明：`defineProps({ name: String })` — JS 和 TS 都支持
> 2. 纯类型声明：`defineProps<{ name: string }>()` — 仅 TS，最简洁
> 3. withDefaults：配合纯类型声明添加默认值
> 4. 引用外部 interface/type

---

### 2. MaybeRef 是什么？为什么有用？

> `MaybeRef<T> = T | Ref<T>`，表示一个值可以是普通值也可以是 Ref。
> 在 Composable 参数中使用，让调用方既可以传静态值也可以传响应式值，内部用 `unref()` 统一取值。

---

### 3. 为什么 reactive 不支持基本类型？

> `reactive` 基于 Proxy，Proxy 只能代理对象。`reactive(0)` 会直接报错。
> 基本类型用 `ref`，或者用 `reactive` 包裹对象。

---

### 4. InjectionKey 有什么作用？

> 为 `provide/inject` 提供类型安全。定义一个 `InjectionKey<T>`，`provide` 时会校验值的类型，`inject` 时自动推断返回类型。没有 InjectionKey 时，inject 只能手动类型断言。

---

## 十六、总结

```
组件类型：
  defineProps<T>()      → Props 类型声明
  defineEmits<T>()      → Emits 类型声明
  defineModel<T>()      → v-model 类型（Vue 3.4+）
  defineSlots<T>()      → 插槽类型（Vue 3.3+）
  defineExpose<T>()     → 暴露方法类型
  InstanceType<typeof>  → 获取组件实例类型

响应式类型：
  Ref<T>               → ref 类型
  MaybeRef<T>          → T | Ref<T>
  ComputedRef<T>       → computed 类型
  unref()              → 取出值
  storeToRefs()        → 结构 store 保持类型

Composable 类型：
  泛型函数 + 明确返回类型 + MaybeRef 入参

环境配置：
  InjectionKey        → provide/inject 类型安全
  .d.ts               → 全局类型声明
  moduleResolution: "bundler"  → Vue 3.3+ 推荐
```

