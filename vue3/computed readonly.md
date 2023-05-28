# computed 
计算属性 可以描述以来响应式状态的复杂逻辑
```
<script setup>
import { reactive, computed } from 'vue'

const author = reactive({
  name: 'John Doe',
  books: [
    'Vue 2 - Advanced Guide',
    'Vue 3 - Basic Guide',
    'Vue 4 - The Mystery'
  ]
})

// 一个计算属性 ref
const publishedBooksMessage = computed(() => {
  return author.books.length > 0 ? 'Yes' : 'No'
})
</script>

<template>
  <p>Has published books:</p>
  <span>{{ publishedBooksMessage }}</span>
</template>
``` 
计算属性值会基于其响应式依赖被缓存, 这也解释了为什么下面的计算属性永远不会更新，因为 Date.now() 并不是一个响应式依赖
```
const now = computed(() => Date.now())
```
computed可以get和set
```
<script setup>
import { ref, computed } from 'vue'

const firstName = ref('John')
const lastName = ref('Doe')

const fullName = computed({
  // getter
  get() {
    return firstName.value + ' ' + lastName.value
  },
  // setter
  set(newValue) {
    // 注意：我们这里使用的是解构赋值语法
    [firstName.value, lastName.value] = newValue.split(' ')
  }
})
</script>
```

# readonly
可以是普通的，可以是响应式的，只读的，深度的。
但是它可以使用 watchEffect观察到