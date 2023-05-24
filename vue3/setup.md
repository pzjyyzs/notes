setup的存在是为了能够使用composition api
```
<template>
    <div>{{ count }}</div>
</template>
<script>
import { ref, reactive } from 'vue';

export default {
    setup() {
        const count = ref(0);
        console.log(count.value);
        return {
            count
        }
    }
}
</script>
```
调用的时机
组件的实例被创建并且初始化props后被调用的， 在```beforeCreate```生命周期前被调用

如果setup返回一个对象(不管是不是响应式)，和被合并到 render函数里面来， 这这个函数是用来渲染模版的，可以在template中直接使用这个对象.
响应式的对象使用```.value```的形式取值的，而setup返回出去，会自动使用.value,在模版中使用就不需要.value。

可以不用template, 使用render函数

```
<template>
  <!-- <div>{{ count }}</div> -->
</template>

<script>
import { ref, h } from 'vue';

export default {
  name: 'HelloWorld',
  props: {
    msg: String,
  },
  setup() {
    const count = ref(0);
    console.log(count.value);
    /* return {
      count,
    }; */
    return () => h('h1', [count.value]);
  },
};
</script>
```
# 参数
setup 有两个参数 第一个是 props 组件的属性。 props是响应式的不需要再```reactive```, 如果需要它变化了 可以使用```watchEffect```和```watch```两个API进行观察。不要结构props的值，这样会丢掉响应式。因为单项数据流的原因，不能在子组件更改 props。

第二个参数是 context 当前组件的执行期上下文 存在2.x中 emit slots 这些api。

https://stackblitz.com/edit/vue-vk5enk?file=src%2Fcomponents%2FHelloWorld.vue,src%2FApp.vue