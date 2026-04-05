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

























