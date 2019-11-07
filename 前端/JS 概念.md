## 执行环境

> 执行环境，有时也叫执行上下文、环境，它定义了变量或者函数有权访问的其他数据，决定了他们各自的行为。

## 作用域

> 作用域是一套用来管理 js 代码在执行环境中根据标识符名称进行变量查找的规则。
>
> js 是词法作用域，由代码写定时确定。

## 闭包

> 闭包是即使离开了创造它的环境，也仍旧可以引用到自由变量的函数，是函数与其相关引用环境组合的实体。

## prototype（[[Prototype]]、\_\_proto\_\_）

> 所有的函数默认都会拥有一个名为 prototype 的公有并且不可枚举的属性，它会指向另一个对象：也叫做他的原型，可以通过名为 prototype 的属性引用来访问它。



**[[Prototype]]**

> JavaScript 中的对象有一个特殊的 [[Prototype]] 内置属性， 其实就是对于其他对象的引用。

```js
const a = { index: 1 };
const b = Object.create(a);
const c = Object.create(b);
// 等价于 c.__proto__.__proto__.index
c.index // 1
```

通过`[[Prototype]]`属性一级一级找，找到 a 处的 index，输出 1，如果找不到则输出 undefined，这条`[[Prototype]]`链就是原型链。

**\_\_proto\_\_**

内置属性，可以修改 prototype 的关联对象，ES6 添加了辅助修改方法 Object.setPrototypeOf。如果想访问原型链，可以通过 .\_\_proto\_\_.\_\_proto\_\_ ... 来访问。

## constructor

```js
function Foo() { }

Foo.prototype.constructor === Foo; // true

var a = new Foo();
a.constructor === Foo; // true
```

> Foo.prototype 默认（在代码中第一行声明时！）有一个公有并且不可枚举的属性 .constructor ， 这个属性引用的是对象关联的函数（本例中是 Foo ）。

