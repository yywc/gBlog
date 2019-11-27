## 前言

javascript 也算是一门面向对象语言，我们经常在开发中使用字面量对量来创建对象，但是自从 ES6 问世以来，我们创建对象的方法多了很多，也更面向对象了一点，下面就分别用 ES5 ES6 创建对象实现继承。

## 1. ES5 创建对象的方式

在 JavaScript 中，创建对象的方式有很多种，最常用的一般是通过字面量的方式，而要创建实例对象则一般通过创建一个构造函数，通过 new 关键字来构造。

虽然 Object 函数和字面量都可以创建对象，但同时也会有一个问题：使用一个接口创建多个对象时，会出现大量重复代码。下面来介绍一些创建对象的变体。

### 1.1 工厂模式

```js
function createPerson(name, age) {
    var obj = new Object();
    obj.name = name;
    obj.age = age;
    obj.sayName = function () {
        console.log(this.name);
    };
    return obj;
}
var person = createPerson('mike', 18);
```

工厂模式解决了创建多个相似对象的问题，但缺点是无法识别对象原型。

这里打印 person 对象，可以看到有 2 个属性和 1 个方法，原型对象是 Obejct，constructor 属性（指向构造函数的指针）指向 Object 对象。

![1](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100732.png)

### 1.2 构造函数

```js
function Person(name, age) {
    this.name = name;
    this.age = age;
    this.sayName = function () {
        console.log(this.name);
    }
}
var person = new Person('mike', 18);
var person2 = new Person('alice', 20);
// 相当于以下操作
var obj = new Object();
obj.__proto__ = Person.prototype;
Person.call(obj, 'mike', 18);
```

![1](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100733.png)

构造函数模式是比较常见的一种方式，通过大写函数名的第一个字母来用以区分普通函数。

构造函数与工厂模式还有以下的不同：

+ 没有显示创建对象
+ 直接将属性赋值给了 this
+ 没有 return

此时创建 person 实例需要通过 new 关键字，通过 new 关键字调用构造函数的过程其实经历了以下四个步骤：

1. 创建一个新对象: var obj = new Object();
2. 将构造函数的原型对象赋值给新的对象 obj: obj.\_\_proto\_\_ = Person.prototype;
3. 执行构造函数中的代码，给新对象 obj 添加属性和方法: Person.call(obj, 'mike', 18);
4. 返回 obj 对象

构造函数解决了工厂模式不能识别实例类型的问题，但是也有一个缺点：在这个例子里它会多次创建了相同函数 sayName。

![1](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100735.png)

### 1.3 原型模式

我们创建每一个函数都有一个 prototype（原型）属性，指向一个对象。这个对象的用途是包含所有特定类型（例子是 Person）的所有实例共享的属性（name age）和方法（sayName）。

```js
function Person() { }
Person.prototype = {
    constructor: Person, // 不指定 constructor 会使 constructor 指向断裂，导致对象类型无法正确识别。
    name: 'mike',
    age: 19,
    hobby: ['football', 'singing'],
    sayName: function () {
        console.log(this.name);
    }
}
var person1 = new Person();
var person2 = new Person();
person1.hobby.push('dancing'); // person2.hobby: ['football', 'singing','dancing']
```

constructor 指向未断裂的情况：指向了 Person

![1](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100738.png)

constructor 指向断裂的情况：失去了 constructor，默认指向了 Object

![1](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100742.png)

原型链示意图：

![1](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100743.png)

下图可见通过原型模式解决了构造函数模式多次创建了 sayName 方法的问题，但聪明的电视机前的你肯定发现了定义的原型属性会被所有的实例共享。

![1](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100747.png)

当我们操作了 person1 的 hobby 对象的时候，person2 的也同时被修改了，这是我们不愿看到的。

### 1.4 组合模式

```js
function Person(name, age) {
    this.name = name;
    this.age = age;
    this.hobby = ['football', 'singing']
}

Person.prototype = {
    constructor: Person, // 不指定 constructor 会使 constructor 指向断裂，导致对象类型无法正确识别。
    sayName: function () {
        console.log(this.name);
    }
}
var person1 = new Person('mike', 18);
person1.hobby.push('dancing');
var person2 = new Person('alice', 19);
```

![1](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100748.png)

通过以上的几种方式的分析，我们差不多也能得到比较好的一种模式了，那就是组合模式。

在构造函数中添加实例属性，在构造函数的原型链上添加实例方法，这样既解决了实例共享，又解决了多次创建相同函数的问题，是目前使用比较广泛的模式。

## 2. ES6 创建对象的方式

ES6 里我们可以通过 class 关键字来定义一个类，class 实际上是一个语法糖，虽然绝大部分的功能可以通过 ES5 实现，但是 class 的写法让对象变的更加清晰，更接近面向对象的语法。
通过 class 来改写组合模式：

```js
class Person {
    constructor(name, age) {
        this.name = name;
        this.age = age;
        this.hobby = ['football', 'singing'];
    }
     sayName() {
         console.log(this.name);
    }
}
var person1 = new Person('mike', 18);
person1.hobby.push('dancing');
var person2 = new Person('alice', 19);
```

![1](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100751.png)

由此对比可见，和 ES5 的结果只有在 \_\_proto\_\_ 对象里的 constructor 显示的是 class，其余的部分都是一致。
通过 babel 编译成 ES5，我们进行一下对比。

```js
'use strict';

var _createClass = function () {
    // 定义属性的配置项
    function defineProperties(target, props) {
        for (var i = 0; i < props.length; i++) {
            var descriptor = props[i];
            descriptor.enumerable = descriptor.enumerable || false;
            descriptor.configurable = true;
            if ("value" in descriptor) descriptor.writable = true;
            Object.defineProperty(target, descriptor.key, descriptor);
        }
    }
    return function (Constructor, protoProps, staticProps) {
        if (protoProps) {
            defineProperties(Constructor.prototype, protoProps);
        }
        if (staticProps) {
            defineProperties(Constructor, staticProps);
        }
        return Constructor;
    };
}();

// 检查实例是否是后者的实例
function _classCallCheck(instance, Constructor) {
    if (!(instance instanceof Constructor)) {
        throw new TypeError("Cannot call a class as a function");
    }
}

var Person = function () {
    function Person(name, age) {
        _classCallCheck(this, Person);

        this.name = name;
        this.age = age;
        this.hobby = ['football', 'singing'];
    }

    // 挂载 sayName 方法
    _createClass(Person, [{
        key: 'sayName',
        value: function sayName() {
            console.log(this.name);
        }
    }]);

    return Person;
}();

var person1 = new Person('mike', 18);
person1.hobby.push('dancing');
var person2 = new Person('alice', 19);
```

抛开对属性的一些配置上的操作，与 ES5 我们所用的组合模式并无不同。

## 3. ES5 实现继承

首先我们通过组合模式创建一个 Animal 父类对象

```js
// 定义一个动物类
function Animal(name) {
    // 属性
    this.name = name || 'Animal';
    // 实例方法
    this.sleep = function () {
        return this.name + ' 正在睡觉!';
    }
}
// 原型方法
Animal.prototype.eat = function (food) {
    return this.name + ' 正在吃: ' + food;
};
```

### 3.1 原型链继承

**核心：** 将父类的实例作为子类的原型（注意不能使用字面量方式定义原型方法，会重写原型链）

```js
function Cat() {}
Cat.prototype = new Animal();
Cat.prototype.name = 'cat';

//　Test Code
var cat = new Cat();
console.log(cat.name); // cat
console.log(cat.eat('fish')); // cat 正在吃：fish
console.log(cat.sleep()); // cat 正在睡觉!
console.log(cat instanceof Animal); // true
console.log(cat instanceof Cat); // true
```

**特点：**

1. 非常纯粹的继承关系，实例是子类的实例，也是父类的实例
2. 父类新增原型方法/原型属性，子类都能访问到
3. 简单，易于实现

**缺点：**

1. 可以在Cat构造函数中，为Cat实例增加实例属性。如果要新增原型属性和方法，则必须放在new Animal()这样的语句之后执行。
2. 无法实现多继承
3. 来自原型对象的引用属性是所有实例共享的
4. 创建子类实例时，无法向父类构造函数传参

**推荐指数**：★★（3、4两大致命缺陷）

### 3.2 构造继承

**核心**：使用父类的构造函数来增强子类实例，等于是复制父类的实例属性给子类（没用到原型）

```js
function Cat(name){
  Animal.call(this);
  this.name = name || 'Tom';
}

// Test Code
var cat = new Cat();
console.log(cat.name); // Tom
console.log(cat.sleep()); // Tom 正在睡觉
// console.log(cat.eat('fish')); // 会报错，原型在这里不可用
console.log(cat instanceof Animal); // false
console.log(cat instanceof Cat); // true
```

**特点：**

1. 解决了原型链继承中，子类实例共享父类引用属性的问题
2. 创建子类实例时，可以向父类传递参数
3. 可以实现多继承（call 多个父类对象）

**缺点：**

1. 实例并不是父类的实例，只是子类的实例
2. 只能继承父类的实例属性和方法，不能继承原型属性/方法
3. 无法实现函数复用，每个子类都有父类实例函数的副本，影响性能

**推荐指数**：★★（缺点3）

### 3.3 实例继承（原型式继承）

**核心**：为父类实例添加新特性，作为子类实例返回

```js
function Cat(name){
  var instance = new Animal();
  instance.name = name || 'Tom';
  return instance;
}

// Test Code
var cat = new Cat();
console.log(cat.name);
console.log(cat.sleep());
console.log(cat instanceof Animal); // true
console.log(cat instanceof Cat); // false
```

**特点**：不限制调用方式，不管是new 子类()还是子类(),返回的对象具有相同的效果

**缺点：**

1. 实例是父类的实例，不是子类的实例
2. 不支持多继承

**推荐指数**：★★

### 3.4 拷贝继承

```js
function Cat(name){
  var animal = new Animal();
  for(var p in animal){
    Cat.prototype[p] = animal[p];
  }
  Cat.prototype.name = name || 'Tom';
}

// Test Code
var cat = new Cat();
console.log(cat.name);
console.log(cat.sleep());
console.log(cat instanceof Animal); // false
console.log(cat instanceof Cat); // true
```

**特点**：支持多继承

**缺点：**

1. 效率较低，内存占用高（因为要拷贝父类的属性）
2. 无法获取父类不可枚举的方法（不可枚举方法，不能使用for in 访问到）

**推荐指数**：★（缺点1）

### 3.5 组合继承

**核心**：通过调用父类构造，继承父类的属性并保留传参的优点，然后通过将父类实例作为子类原型，实现函数复用

```js
function Cat(name){
  Animal.call(this);
  this.name = name || 'Tom';
}
Cat.prototype = new Animal();

// Test Code
var cat = new Cat();
console.log(cat.name);
console.log(cat.sleep());
console.log(cat instanceof Animal); // true
console.log(cat instanceof Cat); // true
```

**特点：**

1. 弥补了方式2的缺陷，可以继承实例属性/方法，也可以继承原型属性/方法
2. 既是子类的实例，也是父类的实例
3. 不存在引用属性共享问题
4. 可传参
5. 函数可复用

**缺点：**
调用了两次父类构造函数，生成了两份实例（子类实例将子类原型上的那份屏蔽了）

**推荐指数**：★★★★（仅仅多消耗了一点内存）

### 3.6 寄生组合继承

**核心**：通过寄生方式，砍掉父类的实例属性，这样，在调用两次父类的构造的时候，就不会初始化两次实例方法/属性，避免的组合继承的缺点

```js
function Cat(name){
  Animal.call(this);
  this.name = name || 'Tom';
}
(function(){
  // 创建一个没有实例方法的类
  var Super = function(){};
  Super.prototype = Animal.prototype;
  //将实例作为子类的原型
  Cat.prototype = new Super();
})();

// 等价于下面这种情况
// function inheritPrototype(sub, sup) {
//    var Fn= function() {}
//    Fn.prototype = sup.prototype;
//    sub.prototype = new Fn();
// }

// inheritPrototype(Cat, Animal);

// Test Code
var cat = new Cat();
console.log(cat.name);
console.log(cat.sleep());
console.log(cat instanceof Animal); // true
console.log(cat instanceof Cat); //true
```

**特点**：堪称完美

**缺点**：实现较为复杂

**推荐指数**：★★★★

## 4. ES6 实现继承

首先还是创建一个 Animal 类

```js
class Animal {
    constructor(name) {
        this.name = name || 'Animal';
        this.sleep = function () {
            return this.name + ' 正在睡觉!';
        }
    }
    eat(food) {
        return this.name + ' 正在吃: ' + food;
    };
}
```

然后通过 extends 关键字来继承 Animal

```js
class Cat extends Animal {
    constructor(name, age) {
        super(name);
        this.age = age; // 新增的子类属性
    }
    eat(food) {
        const result = super.eat(food); // 通过 super 调用父类方法
        return this.age + ' 岁的 ' + result;
    }
}
const cat = new Cat('miao', 3);
```

![1](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100758.png)

## 总结

总的来说，ES6 的 class 语法糖更清晰和优雅地实现了创建对象和对象继承。
但是我们要想更好的理解 class，那么关于 ES5 的对象、对象继承以及原型链等知识也是要掌握的很牢固。
