# 经典题

直接看经典面试题，没做对说明对应核心原理没掌握透。

## 数据类型

```javascript
let str = "hello"
str[0] = "H"

console.log(str) 
// hello
```

```javascript
function change(a) {
  a = 20
}

let x = 10

change(x)

console.log(x) // 10
```

```javascript
function change(obj) {
  obj.name = "Jerry"
}

let user = { name: "Tom" }

change(user)

console.log(user.name) // Jerry
```

```javascript
function change(obj) {
  obj = { name: "Jerry" }
}

let user = { name: "Tom" }

change(user)

console.log(user.name) // Tom
```

```javascript
let a = [1,2,3]

let b = a

b = [4,5,6]

console.log(a) // [1,2,3]
```

```javascript
let obj = { a:1, b:{c:2} }

let copy = {...obj}

copy.b.c = 3

console.log(obj.b.c) // 3
```

```javascript
[] == false // true

// false → 0

// [] → "" → 0

// 0 == 0
```
