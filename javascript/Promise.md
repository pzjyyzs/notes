# 解决了什么问题

## 回调的问题

```
doA(function(){
	doC();
	doD(function(){
		doF();
	})
	doE();
});
doB();
```
这段代码的执行顺序是A->B->C->D->E->F,你需要不停的上下移动视线；如果A或D不一定是异步的呢，这里的顺序就有可能改变；跟踪异步流变得困难。如果嵌套过多，我们无法很好的跟踪异步代码的执行过程。我们顺序阻塞式的大脑计划行为无法很好地映射到面向回调的异步代码。

信任问题。如果你使用了第三方的插件，你传入一个回调函数，由第三方来执行这个回调，这种控制的转移会带来一系列信任问题，比如它回调次数过多怎么办。

## Promise
Promise解决了信任问题，我们需要第三方提供 任何何时结束的能力，然后由我们自己决定接下来需要干些什么。

我们使用 new Promise 来创建一个Promise,它接受一个函数作为参数，函数有两个参数，第一个参数代表标识完成，第二代表标识拒绝。
```
const p1 = new Promise((reslove, reject) => {
	setTimeout(() => {
		reslove('1')
	}, 100)
})
p1.then((val) => {
	console.log(val)
}, () => {

})
```

你可以在p1对象的then方法，在标识完成或者拒绝后做些什么。then函数第一个参数，接受完成所执行的内容，第二个参数接受拒绝所执行的内容。

## 链式流

```
var p = Promise.resolve(21);
p.then(function(v){
	console.log(v);
	return v * 2
}).then(function(v){
	console.log(v)
})
```
这可以一直任意扩展下去，只要保持把先前的then链接到自动创建的每一个Promise即可。

```
var p = Promise.reslove(21);
p.then(function(v){
	console.log(v);
	return new Promise(function(reslove, reject){
		reslove(v * 2);
	})
}).then(function(v){
	console.log(v)
})
```
返回Promise也会进行决议

## 其他方法

### catch
如果有reject函数用catch可以捕捉到链式调用中所有的错误。如果有reject函数则不会进入catch,如果有语法错误会进入catch。这里有一个问题 如果catch中有错误，要去哪里捕获呢
```
var p = Promise.reject(12);

p.then(() => {}, (msg) => {
	console.log('then', msg);
}).catch(msg => {
	console.log('catch', msg)
})

var p = Promise.resolve(12);

p.then((msg) => {
	console.log(msg.toLowerCase())
}).catch(msg => {
	console.log('catch', msg)
})
```

### Promise.all([])
需要一个有Promise实例组成的数组，等待数组中的所有Promise完成，返回一个由所有传入promise的完成消息组成的数组，与指定的顺序一致（与完成顺序无关）。如果当中任何一个Promise被拒绝的话，主Promise.all([...]) promise就会立刻被拒绝并丢弃来自其他所有promise的全部结果。

### Promise.race([])
需要一个有Promise实例组成的数组，只等待第一个完成的，抛弃其他的Promise

## 局限性

- Promise链中的错误很容易被无意中默默忽略掉
- 无法从外部停止他的进程
- 内部抛出的错误无法返回外部

# 手写
promiseaplus.com

# 生成器

前提 
	generator 生成器函数 -> 生成一个东西的函数 -> function *函数名 () {}
	iterator 迭代器对象
	yield 产出一个值 暂停标识
```
function *generator() {
	yield '123';
	yield '姓名：34'
}

const iterator = generator();
let res = iterator.next();
res = iterator.next();
// res { value: 123, done: false }
```
生成器返回一个迭代器, 迭代器可以使用 next() 获得一个对象，对象有value和done, value是yield返回的值，done代表是否返回结束。

## 异步迭代

代码
```
const fs = require('fs').promises;

  

function *getUserClasses (uid) {

let userDatas = yield fs.readFile('./data/user.json');

userData = JSON.parse(userData);

const userData = userDatas.find(user => user.id === uid);

let classDatas = yield fs.readFile('./data/class.json');

classDatas = JSON.parse(classDatas);

  

let userClassData = {

id: userData.id,

name: userData.name,

classes: []

}

  

classDatas.map(c => {

const studentsArr = JSON.parse.parse(c.students);

studentsArr.map(s => {

if (s === uid) {

userClassData.classes.push({

id: c.id,

name: c.name

})

}

})

})

  

return userClassData

}

  

const it = getUserClasses(1);

console.log('123', it.next())
```
执行到yield暂停，调用next才赋值，再执行到yield。
如果需要拿到所有结果需要
```
const it = getUserClasses(1);

const { value, done } = it.next();

value.then(res => {

console.log(res.toString('utf-8'));

const { value, done } = it.next(res);

console.log(value)

})
```
 需要嵌套调用比较麻烦。
 可以使用co库，来异步迭代，这也是async，await的用法
```
function co(iterator) {

return new Promise((resolve, reject) => {

function walk(data) {

const { value, done } = iterator.next(data);

if (!done) {

Promise.resolve(value).then(res => {

walk(res);

}, reject)

} else {

resolve(value)

}
co(getUserClasses(1)).then(res => {

console.log(res)

}).catch((err) => {

console.log(err);

})

}

  

walk()

})

}

//async await
async function getUserClasses(uid) {

let userDatas = await fs.readFile('./data/user.json');

userDatas = JSON.parse(userDatas);

const userData = userDatas.find(user => user.id === uid);

let classDatas = await fs.readFile('./data/class.json');

classDatas = JSON.parse(classDatas);

  

let userClassData = {

id: userData.id,

name: userData.name,

classes: []

}

  

classDatas.map(c => {

const studentsArr = JSON.parse(c.students);

studentsArr.map(s => {

if (s === uid) {

userClassData.classes.push({

id: c.id,

name: c.name

})

}

})

})

  

return userClassData

}

  

getUserClasses(1).then((res) => {

console.log(res)

}).catch(error => {

console.log('error', error)

})
```
await只在async函数中使用，await后面接的是Promise函数，若是普通函数，则无区别。
错误处理
```
const response = await axios.get('/xxx').then(null, errorHandle)
console.log(response);
```
成功处理在左边，错误处理在右边。
await 在for循环里是串行的(一个接着一个)，在forEach循环里是并行的(同时做)