```
import  { createApp } from 'vue';
import App from './App.vue';

const app = createApp(App);

console.log(app.config)
app.mount('#app');
```

App 是一个对象，放到createApp后，有一个组件实例并挂载到DOM对象上。
config有一些属性

# errorHandler warnHandler
vue运行过程中的错误或警告，可以在config的这两个函数中捕获。不常用

# globalProperties
添加一个全局的对象，所有在这个组件实例的子组件都可以访问到
```
import  { createApp } from 'vue';
import App from './App.vue';
import * as utils from './libs/utils';

const app = createApp(App);

app.config.globalProperties.utils = utils;
app.mount('#app');

// 使用
const { ctx } = getCurrentInstance();
ctx.utils
```
内部属性如果名字和全局属性相同，内部优先。

# isCustomElement
如果使用自定义标签，vue会提出一个警告
```
app.config.isCustomElement = (tag) => {
    return tag.startsWith('test'); // 以test开头
}
```

