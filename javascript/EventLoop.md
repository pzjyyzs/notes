[文章](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick)

# 进程

- cpu正在进行的一个任务的运行过程的调度单位
- 浏览是一个多进程程序
- 进程是计算机调度的基本单位
- 进程包含线程，线程在进程中运行
- 每一个浏览器tab都会开启一个进程(相互之间不影响)
- 浏览器有一个主进程(用户界面)
- 每一个tab各自有独立的渲染进程(浏览器内核 Renderer,渲染引擎)，网络进程(网络请求)，gpu进程(动画与3d绘制)，插件进程
## 渲染进程

GUI渲染与JS引擎线程运行互斥
### GUI渲染线程

- 解析HTML,CSS
- 构建DOM/Render树
- 初始布局与绘制
- 重绘与回流
### JS引擎线程
一个主线程与多个辅助线程配合
一个浏览器只有一个JS引擎，事件环在这个环节。
解析JS脚本
运行JS代码
JS有可能改变DOM,所以GUI线程要等JS空闲的时候才更新

宏任务： 用户交互事件 如：click,setTImeout,Ajax
微任务： Promise.then, MutationObserver
## 整体思路
JS执行栈
先执行 同步代码

有微任务会把微任务放到微任务队列，
有宏任务会把回调放到宏任务队列，
同步代码执行完成后，
JS执行栈会先执行微任务队列里的回调，等到微任务队列执行完毕，
进行GUI渲染，
之后在JS执行栈选择宏任务中的一个回调，执行哪一个看宏任务是否触发，例如setTImeout事件是否足够，Ajax是否返回，用户是否触发事件，如果有多个，则执行第一个。如果回调里有微任务，则加入微任务队列，执行完宏任务后再执行微任务，再进行GUI渲染，在执行后面的宏任务，以此循环直到所有事件都执行完毕。

```document.body.style.backgroundColor = 'orange';

console.log(1);

  

setTimeout(() => {

document.body.style.backgroundColor = 'red';

console.log(2)

}, 100);

  

Promise.resolve(3).then((num) => {

document.body.style.backgroundColor = 'purple';

console.log(num)

})

  

console.log(4)

结果
bgcolor: orange => 没渲染 还没到GUI渲染就被改变了
1
4
bgcolor: purple => 渲染
3
bgcolor: red => 没渲染
2
bgcolor: red => 渲染了
```

```
Promise.resolve().then(() => {
	console.log('1');
	setTimeout(() => {
		console.log('2')
	}, 0)
})

setTimeout(() => {
	console.log('3');
	Promise.reslove().then(() => {
		console.log('4')
	})
}, 0)

// 1 3 4 2
```



