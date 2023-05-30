# watchEffects
首次会立刻执行，然后响应式的追踪, 默认在所有组件更新前执行。 依赖收集的功能。在组件卸载的时候自动停止。他会返回一个函数，可以手动的停止。

watchEffect
第一个参数 接受一个回调函数，该回调函数用于执行响应后的操作。回调函数有一个参数也是一个函数，用来注册清理回调。清理回调在该副作用的下一次执行前被调用，可以用来清理无效的副作用，例如等待中的异步请求。

```
watchEffect(async (onCleanup) => {
  const { response, cancel } = doAsyncWork(id.value)
  // `cancel` 会在 `id` 更改时调用
  // 以便取消之前
  // 未完成的请求
  onCleanup(cancel)
  data.value = await response
})
```

为什么不像 React useEffect 一样使用 返回值作为取消请求的方法。
如果这个回调函数是一个异步的函数，useEffect是无法接受async异步函数的，因为 await 会默认返回一个Promise,会和取消副作用的函数冲突

```
watchEffect(async (onInvalidate) => {
    const data = await getData();
    onInvalidate(() => {

    });
})
```

第二参数 是一个可选项， 可以用来调整副作用的刷新时机或调试副作用的依赖
flush 刷新时机 默认是 'pre' 组件刷新前，'post' 组件刷新后 'sync' 同步执行的 效率低，少用。

只会在开发模式下运行
onTrack 函数 第一次也会执行 调试使用
onTrigger 函数 改变才会执行 调试使用

Note: watchEffect 第一次是在组件刷新前的， 如果想访问DOM或者 Template ref 可以在 onMounted中使用 watchEffect

# watch
对比 watchEffect
- 需要指定一个需要观察的值，等它改变了才执行；
- 更明确什么情况下才会触发；
- 有之前的值和当前的值；
```
const count = ref(0);
watch(() => {
    return count.value
}, (newValue, oldValue) => {
    console.log(newValue, oldValue)
})
```
如果要观察的是ref,可以直接使用不需要回调函数，如果是reactive就必须回调函数。
```
watch(count, () => {

})
```

批量监听
```
    const age = ref(0);
    const name = ref('zs');

    setTimeout(() => {
      age.value = 1;
      name.value = 'ls';
    }, 1000);

    watch(
      () => {
        return [age.value, name.value]; //  
      },
      ([newAge, newName], [oldAge, oldName]) => {
        console.log('watch', newAge, newName);
        console.log('watch', oldAge, oldName);
      }
    );
    // ref 可以这样写
     watch([age, name],
      ([newAge, newName], [oldAge, oldName]) => {
        console.log('watch', newAge, newName);
        console.log('watch', oldAge, oldName);
      }
    );
```

watc和watchEffect 一样，在第三个参数可以选择调用时机和debugging, 第二个回调函数的第三个参数 是事件可以手动停止跟踪。

https://stackblitz.com/edit/vue-eywdhd?file=src%2FApp.vue,src%2Fcomponents%2FCustom.vue,src%2Fcomponents%2FEffect.vue,src%2Fcomponents%2FHelloWorld.vue