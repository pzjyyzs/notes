# promise 异步问题同步化的解决方案

promise要解决的问题是什么，它为什么构造函数是同步的，then函数却是异步的。
来看这段代码
```
data.json
[
    {
        "id": 1,
        "name": "zhangsan"
    },
    {
        "id": 2,
        "name": "lisi"
    },
    {
        "id": 3,
        "name": "wangwu"
    }
]

index.js
// 异步程序
$.ajax({
    url: "http://localhost:3000/data.json",
    success (data) {
        console.log(data.map(item=>item.name));
    }
});
console.log("test");

```

这里是先打印 test , 再执行ajax，执行完成后会执行success中的方法;
这个时候，我想success 中的方法抽到外面来呢，这就需要添加一个async属性
```
const data = $.ajax({
    url: "http://localhost:3000/data.json",
    async: false // 同步
});
console.log(data.responseJSON.map(item=>item.name));
console.log("test");

```
这里相当于是同步的代码，不仅阻塞了map，同时也阻塞了test。

promise能很好的解决这个问题，不阻塞其他代码
```
const p = new Promise((resolve, reject) => {
    $.ajax({
        url: 'http://localhost:3000/data.json',
        success (data) {
            resolve(data);
        }
    })
})
p.then(res => {
    console.log(res.map(item=>item.name));
})
console.log("test");
```