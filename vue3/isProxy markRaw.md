# isProxy 
判断这个proxy对象是不是 reactive 或 readonly 创建的

# isReactive
判断这个proxy对象是不是 reactive 创建的.
如果使用reactive创建了一个对象，在使用readonly 包裹，它也会返回 true

# isReadonly
判断这个proxy对象是不是 readonly 创建的.

# shallowReactive
只用第一层才是响应式的

# shallowReadonly
只有第一层才是readonly的

# toRaw
从reactive 和 readonly 返回一个普通对象；
```
const obj = { a: 1, b: 2 }
const proxyObj = reactive(obj);
const newObj = toRaw(proxyObj);
newObj === obj // true
```

# markRaw
标记一个对象，他不会转换一个proxy对象
```
const foo = markRaw({})
console.log(isReactive(reactive(foo)));

const bar = reactive({ foo });
console.log(isReactive(bar.foo))l
```

markRaw() 和类似 shallowReactive() 这样的浅层式 API 使你可以有选择地避开默认的深度响应/只读转换，并在状态关系谱中嵌入原始的、非代理的对象。它们可能出于各种各样的原因被使用：
- 有些值不应该是响应式的，例如复杂的第三方类实例或 Vue 组件对象。

- 当呈现带有不可变数据源的大型列表时，跳过代理转换可以提高性能。

```
const foo = markRaw({
  nested: {}
})

const bar = reactive({
  // 尽管 `foo` 被标记为了原始对象，但 foo.nested 却没有
  nested: foo.nested
})

console.log(foo.nested === bar.nested) // false
```