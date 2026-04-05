# this

## 默认绑定

```javascript
function foo() {
    console.log(this);
}

foo(); // 浏览器: window, 严格模式: undefined
```

## 隐式绑定

调用函数时，函数前面的点 . 所在对象就是 this 的指向

```javascript
const obj = {
    name: 'Alice',
    greet() {
        console.log(this.name);
    }
};

obj.greet(); // Alice
```

链式调用时只绑定最后一个对象.
```javascript
const obj2 = { a: { b: function() { console.log(this); } } };
obj2.a.b(); // 输出 {b: ƒ}
```

## 显式绑定

```javascript
function greet() {
    console.log(this.name);
}

const person = { name: 'Bob' };

greet.call(person);  // Bob
greet.apply(person); // Bob

const boundGreet = greet.bind(person);
boundGreet(); // Bob
```

## new 绑定（Constructor Binding）

```javascript

function Person(name) {
    this.name = name;
}
const p = new Person('Charlie');
console.log(p.name); // Charlie
```
规则：使用 new 调用函数时：

- 创建一个新对象。
- this 指向这个新对象。
- 如果函数没有返回对象，则返回新对象。

## 箭头函数 this

箭头函数没有自己的 this，它的 this 是在定义时从外层作用域继承的。
```javascript
const obj = {
    name: 'Alice',
    greet: () => {
        console.log(this.name);
    }
};
obj.greet(); // undefined (箭头函数的 this 绑定到外层，全局或模块作用域)
```



## 经典题

普通函数调用 → 默认绑定。
```javascript
var a = 10;

function foo() {
  console.log(this.a);
}

foo(); // 10
```

隐式绑定：this = 调用该函数的对象
```javascript
var a = 10;

const obj = {
  a: 20,
  foo: function () {
    console.log(this.a); 
  }
};

obj.foo(); // 20
```

隐式绑定丢失: 函数引用赋值。
```javascript
var a = 10;

const obj = {
  a: 20,
  foo: function () {
    console.log(this.a);
  }
};

const bar = obj.foo;

bar(); // 10
```


```javascript
var name = "global";

const obj = {
  name: "obj",
  foo: function () {
    console.log(this.name);
  }
};

const obj2 = {
  name: "obj2",
  foo: obj.foo
};

// 跟着调用对象走，是 obj2
obj2.foo(); //  "obj2",
```

```javascript
var a = 10;

function foo() {
  console.log(this.a);
}

foo.call({ a: 20 }); // 20
```


bind: 返回一个永久绑定 this 的新函数.
相当于

> bar = function(){
>   foo.call(obj)
> }

```javascript
function foo() {
  console.log(this.a);
}

const obj = { a: 10 };

const bar = foo.bind(obj);

bar.call({ a: 20 }); // 10
```

this优先级：new > bind

```javascript
function Foo(a) {
  this.a = a;
}

const obj = {};

const bar = Foo.bind(obj);

const baz = new bar(10);

console.log(obj.a); // undefined
console.log(baz.a); // 10
```

箭头函数+闭包

```javascript
var name = "global";

const obj = {
  name: "obj",
  foo: function () {
    return () => {
      console.log(this.name);
    };
  }
};

const fn = obj.foo();

fn(); // obj
```


```javascript
var name = "global";

const obj = {
  name: "obj",
  foo: function () {
    console.log(this.name);

    return function () {
      console.log(this.name);
    };
  }
};

const bar = obj.foo(); // obj

bar(); // global
```

## 优先级


```
new
  ↓
bind
  ↓
call / apply
  ↓
隐式绑定（obj.fn()）
  ↓
默认绑定
```
