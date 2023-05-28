# reactive
接受对象， 使用proxy 包装后的对象, 深度的响应式对象，无论多少层
```
<template>
  <h1>{{ count }}</h1>
</template>

<script>
import { reactive } from 'vue';
export default {
  name: 'HelloWorld',
  props: {
    msg: String,
  },
  setup() {
    const obj = reactive({ count: 0 });
    return {
      count: obj.count,
    };
  },
};
</script>
```

# ref 
有一个value值，返回一个可变的响应式对象，只有一个属性```.value``` 可以访问和修改值。如果传递一个object,会使用reactive创建一个深度的响应式数据。在
return里面不需要.value。

- 如果在reactive 中使用 ref 使用时不需要.value
- 如果是reactive中的ref, 新的ref会覆盖旧的ref.
- 如果是reactive中的ref,在Map或者Array中,则ref不会自动展开，需要.value

# unref
使用这个 会判断你传过来的值是不是一个响应式的值，如果是返回 .value,如果不是则返回对象本身。

# toRef
把响应式数据的其中一项属性进行单独的提取出来进行，保持一个响应式的到原对象。如果有其他componsition api需要用到某个属性，可以保持响应式。
```
<script>
const state = reactive({
    name: 'test',
    age: 12
});
const nameRef = toRef(state, 'name');
nameRef.value = 'change';
console.log(state.name);

state.name = 'test';
console.log(nameRef.value);
<script>
```

# toRefs
转化一个响应式的对象成一个普通的对象，普通的对象的属性具有响应式，保持响应式到响应式数据。
```
<script>
export default {
    name: 'App',
    steup() {
        const state = reactive({
            name: '张三',
            age: 30
        });
        const stateRefs = toRefs(state);
        console.log({ ...stateRefs });
        console.log(isRef(stateRefs, stateRefs.name))
        return {
            ...stateRefs // 这样可以不用state.name 直接name
        }
    }
}
</script>
```

# isRef
判断一个值是不是 ref


https://stackblitz.com/edit/vue-eywdhd?file=src%2Fcomponents%2FHelloWorld.vue
