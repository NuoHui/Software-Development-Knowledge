# Promise

Promise 本质是一个 有限状态机（Finite State Machine）。状态一旦改变，不可逆。

```
pending
fulfilled
rejected

pending → fulfilled
pending → rejected
```

```javascript
const p = new Promise((resolve, reject) => {
  resolve(1)
  reject(2)
})

p.then(console.log) // 1
```

## thenable

只要对象具有 then 方法就叫做 thenable。
如果 resolve 的值是 thenable，Promise 会继续解析它。

```javascript
then()

const obj = {
  then(resolve) {
    resolve(10)
  }
}

Promise.resolve(obj).then(console.log) // 10
```

## 值穿透（Value Passing）

```javascript
Promise.resolve(1)
  .then()
  .then()
  .then(console.log) // 1

// value => value
```

当 then 没有传入函数时，值自动向下传递。


## 经典题

```javascript
const p = new Promise((resolve, reject) => {
  resolve(1)
})

p.then(res => {
  console.log(res)
})

console.log(2)
// 2 1
// 同步代码先执行，then 方法属于微任务
```

```javascript
Promise.resolve(1)
  .then(res => {
    return res + 1
  })
  .then(res => {
    console.log(res) // 2
  })
```

```javascript
Promise.resolve({
  then(resolve) {
    resolve(100)
  }
}).then(console.log) // 100
```


```javascript
console.log(1)

setTimeout(() => {
  console.log(2)
})

Promise.resolve()
  .then(() => console.log(3))
  .then(() => console.log(4))

console.log(5)
// 1 5 3 4 2
```

我们把这道 Promise 终极题一步一步拆解。核心要理解三个机制：

- then 回调属于 microtask（微任务）
- 微任务 FIFO 队列
- 如果 then 返回 Promise，会产生新的微任务。

```javascript
Promise.resolve()
  .then(() => {
    console.log(1)
//   return Promise.resolve() 会产生新的微任务
    return Promise.resolve(2)
  })
  .then(res => {
    console.log(res)
  })

Promise.resolve()
  .then(() => {
    console.log(3)
  })

// 1 3 2
```
当 then 返回 Promise：不会立即执行下一个 then。而是生成新的 microtask，排到队列末尾。

```javascript
Promise.resolve()
  .then(() => {
    console.log(1)
  })
  .then(() => {
    console.log(2)
  })

Promise.resolve()
  .then(() => {
    console.log(3)
  })
  .then(() => {
    console.log(4)
  })

// 1 3 2 4
```
这里流程是这样的：
```javascript
then 的回调是微任务
then 链不会一次性进入队列
只有当前 then 执行完
才会把下一个 then 放入微任务队列

then 是微任务
then 链逐个入队
```


## 事件循环调度

```
Call Stack（调用栈）
        │
        ▼
同步代码执行完
        │
        ▼
Microtask Queue（微任务）
Promise.then
Promise.catch
Promise.finally
queueMicrotask
MutationObserver
        │
        ▼
Macrotask Queue（宏任务）
setTimeout
setInterval
setImmediate(Node)
I/O
UI render
```

执行顺序：

```
同步任务
↓
清空 microtask queue
↓
执行一个 macrotask
↓
再次清空 microtask
↓
循环
```

```javascript

async function foo() {
  console.log(1)
  // await 相当于 Promise.resolve().then(...)
  await Promise.resolve()

  console.log(2)
}

console.log(3)

foo()

console.log(4)

// 3 1  4 2
```

















