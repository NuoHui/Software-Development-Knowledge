# 经典题

直接看经典面试题，没做对说明对应核心原理没掌握透。


## 作用域

```javascript
var a = 10

function foo() {
  console.log(a)
}

function bar() {
  var a = 20
  foo()
}

bar() // 10
```

```javascript
if (true) {
  let a = 1
}

console.log(a) // ReferenceError
```

| 特性   | var       | let  | const |
| ---- | --------- | ---- | ----- |
| 作用域  | 函数        | 块    | 块     |
| 变量提升 | 是         | 是    | 是     |
| 初始化  | undefined | 不初始化 | 不初始化  |
| TDZ  | 无         | 有    | 有     |
| 重复声明 | 可以        | 不行   | 不行    |

```javascript
var a = 1

function foo() {
  console.log(a)
  var a = 2
}

foo() // undefined
```

```javascript
let a = 1

{
  console.log(a) // ReferenceError TDZ
  let a = 2
}
```
