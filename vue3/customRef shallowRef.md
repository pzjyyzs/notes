# customRef
创建一个自定义的ref;

```
<template>
  <span>{{ text }}</span>
  <input type="text" v-model="text" />
</template>

<script>
import { customRef } from 'vue';
function useDebounce(val, delay = 200) {
  let t = null;
  return customRef((track, trigger) => {
    return {
      get() {
        track();
        return val;
      },
      set(newValue) {
        clearTimeout(t);
        t = setTimeout(() => {
          val = newValue;
          trigger();
        }, delay);
      },
    };
  });
}
export default {
  name: 'Custom',
  setup() {
    const text = useDebounce('');
    return {
      text,
    };
  },
};
</script>

<style scoped></style>

```
# shallowRef
```
setup() {
    const val = ref({
      name: 'test',
    });
    val.value = {
      age: 12,
    };
    console.log(val.value);
    const shall = shallowRef({
      val: '123',
    });
    shall.value = {
      age: 13,
    };
    console.log(shall.value);
    return {
      text,
    };
  },
```
ref 替换了value后，显示的是一个Proxy对象。
shallowRef 替换了value后是一个 普通的对象

# triggerRef
变成普通对象后，属性值的改变是无法追踪的，这个时候可以使用triggerRef 手动的追踪值变化。
```
const shallow = shallowRef({
  greet: 'hello, world'
});

watchEffect(() => {
  console.log(shallow.value.greet);
});

shallow.value.greet = 'Hello, universe'

triggerRef(shallow);
```

https://stackblitz.com/edit/vue-eywdhd?file=src%2FApp.vue,src%2Fcomponents%2FCustom.vue,src%2Fcomponents%2FHelloWorld.vue