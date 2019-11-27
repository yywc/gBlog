## 前言

在 JavaScript 中，对象可谓是一个非常重要的知识点。什么原型链啊，拷贝啊，继承啊，创建啊等等等等。在我之前的文章中已经对对象的创建和继承做了一个简单的介绍，[【JavaScript】ES5/ES6 创建对象与继承](https://juejin.im/post/5b3b31586fb9a04fb614c0a9)，那么这篇文章主要是针对对象的拷贝。

[2018-07-31更新: 循环引用以及包装对象拷贝](https://juejin.im/post/5b5d3f54e51d453467551604#heading-10)

## 1. 拷贝前的准备

我们先定义一个构造函数，创建好一个等待拷贝的对象。以下操作不考虑循环引用、Date 对象以及 RegExp 对象的拷贝等问题。

```js
function Person(name, age, job, ) {
    this.name = name
    this.age = age
    this.job = job
    this.height = function () { }
    this.weight = Symbol.for('weight')
    this.friend = {
        name: 'kangkan',
        age: 15
    }
}

Person.prototype.hobby = function () {
    return ['football', 'basketball']
}

const person = new Person('mike', null, undefined)
```

## 2. 浅拷贝

对象不同于 Number、String 等基础类型，它是一个引用类型，也就说它的值是保存在堆上，通过内存地址来访问的。简单来看

```js
const a = {one: 1}
const b = {one: 1}
a === b // false
```

如果 obejct1 的引用地址和 object2 一致，那么这就是浅拷贝，实现方式有三种。

### 2.1 直接赋值

```js
const a = {one: 1}
const b = a
b === a // true
a.two = 2
console.log(b.two) // 2
```

### 2.2 遍历拷贝

```js
const simpleClone = function (target) {
    if (typeof target !== 'object') {
        throw new TypeError('arguments must be a Object!')
    }
    let obj = {}
    // 设置原型
    const prototype = Reflect.getPrototypeOf(target)
    Reflect.setPrototypeOf(obj, prototype)
    // 设置属性
    Reflect.ownKeys(target).forEach((key) => {
        obj[key] = target[key]
    })
    return obj
}
const clonePerson = simpleClone(person)
```

可以看出拷贝的结果还是令人满意的。

下图 Object.assign(person) 应为 Object.assign({}, person)

![遍历拷贝](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100819.png)

### 2.3 Object.assign(target, source)

通过这个方法也能达到相同的效果

> const simpleClonePerson = Object.assign({}, person)

### 2.4 扩展运算符

> const simpleClonePerson = {...person}

![扩展运算符](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100831.png)

但是这里有个问题，原型对象丢失了。无法判断 simpleClonePerson 的实例。

但是操作一下 clonePerson.friend 对象，给它添加一个属性就会发现，person 对应的也增加了一个新属性。这不是我们的预期。

也就说通过 simpleClone 和 Object.assign 拷贝的对象只有第一层是深拷贝，第二层就是浅拷贝了。是对引用地址的拷贝。

## 3. 深拷贝

简单来说，以上的浅拷贝方法，在对象深度只有一层的时候其实就是深拷贝。但是当对象的深度大于1，那么对象里面的对象就无法完成深拷贝了。

深拷贝的方法也有两种。

### 3.1 利用 JSON

> const clonePerson = JSON.parse(JSON.stringify(person))

![JSON](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100842.png)

从图中也能看出来，利用 JSON 的方法也是会有很多缺点的。

缺点1：会忽略 undefined

缺点2：不能序列化函数

缺点3：无法拷贝 Symbol

### 3.2 递归拷贝

递归拷贝其实也就是在浅拷贝的遍历拷贝上新增了一些东西

```js
const deepClone = function (target) {
    if (typeof target !== 'object') {
        throw new TypeError('arguments must be a Object!')
    }
    let obj = {}
    // 设置原型
    const prototype = Reflect.getPrototypeOf(target)
    Reflect.setPrototypeOf(obj, prototype)
    // 设置属性
    Reflect.ownKeys(target).forEach((key) => {
        const value = target[key]
        if (value !== null && typeof value === 'object') {
            obj[key] = deepClone(value)
        } else {
            obj[key] = value
        }
    })
    return obj
}
```

![递归拷贝](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100849.png)

达到了想要的效果。

## 4. 补充

### 4.1 关于 Date、RegExp 对象的拷贝

我们扩展一下 Person 构造函数

```js
function Person(name, age, job, ) {
    this.name = name
    this.age = age
    this.job = job
    this.height = function () { }
    this.weight = Symbol.for('weight')
    this.friend = {
        name: 'kangkan',
        age: 15
    }
    this.family = new Person2()
    this.date = new Date('2018-06-06')
    this.regExp = /test/ig
}

function Person2() { }
```

可以看到这里就多了一个 date 属性和 regExp 属性，如果通过之前普通的 deepClone 的话，会出现如下结果。

![拷贝包装对象](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100851.png)

所以我们需要对 deepClone 方法进行一定的改造

```js
const deepClone = function (target) {
    if (typeof target !== 'object') {
        throw new TypeError('arguments must be a Object!')
    }
    let obj = {}
    // 设置原型
    const prototype = Reflect.getPrototypeOf(target)
    Reflect.setPrototypeOf(obj, prototype)
    // 设置属性
    Reflect.ownKeys(target).forEach((key) => {
        const value = target[key]
        // 在此处进行改造
        try {
            const Constructor = Reflect.getPrototypeOf(value).constructor
            // 这里只针对 Date 对象和 RegExp 对象进行简单的说明
            if (Constructor === Date || Constructor === RegExp) {
                obj[key] = new Constructor(value.valueOf())
            } else {
                obj[key] = deepClone(value)
            }
        } catch (e) {
            obj[key] = value
        }
    })
    return obj
}
```

我们再来看看打印结果

![准备拷贝包装对象](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100857.png)

### 4.2 关于循环引用的问题简述

```js
person.family = person // 此处出现循环引用
const deepClone = function (target) {
    if (typeof target !== 'object') {
        throw new TypeError('arguments must be a Object!')
    }
    let obj = {}
    // 设置原型
    const prototype = Reflect.getPrototypeOf(target)
    Reflect.setPrototypeOf(obj, prototype)
    // 设置属性
    Reflect.ownKeys(target).forEach((key) => {
        const value = target[key]
        try {
            const Constructor = Reflect.getPrototypeOf(value).constructor
            if (Constructor === Date || Constructor === RegExp) {
                obj[key] = new Constructor(value.valueOf())
            } else {
                obj[key] = deepClone(value)
            }
        } catch (e) {
            obj[key] = value
        }
    })
    return obj
}
```

![循环引用](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100909.png)

由上图可以看到，通过 deepClone 方法进行深拷贝，一旦出现循环引用会导致栈溢出。

我们需要对 deepClone 方法再次进行改造

```js
const deepClone = function (target) {
    if (typeof target !== 'object') {
        throw new TypeError('arguments must be a Object!')
    }
    // 已经访问过的对象集合
    const visitedObjs = []
    // 克隆的对象集合
    const clonedObjs = []
    const clone = function (source) {
        if (visitedObjs.indexOf(source) === -1) { // 这里是判断该原对象是否被访问过
            visitedObjs.push(source) // 放入数组中
            const obj = {} // 创建一个待克隆的新对象
            // 设置原型
            const prototype = Reflect.getPrototypeOf(source)
            Reflect.setPrototypeOf(obj, prototype)
            clonedObjs.push(obj); // 将其置入克隆对象集合中
            // 设置属性
            Reflect.ownKeys(source).forEach((key) => {
                const value = source[key]
                try {
                    const Constructor = Reflect.getPrototypeOf(value).constructor
                    if (Constructor === Date || Constructor === RegExp) {
                        obj[key] = new Constructor(value.valueOf())
                    } else {
                        obj[key] = clone(value) // 此处不能再递归调用 deepClone，而是要改为 clone 方法
                    }
                } catch (e) {
                    obj[key] = value
                }
            })
            return obj
        } else {
            // 如果该对象已经被访问过了，则直接从克隆对象中返回。返回的对象的索引是 source 在 visitedObjs 中的索引位置。
            return clonedObjs[visitedObjs.indexOf(source)]
        }
    }
    return clone(target)
}
```

再来看看效果

![循环引用拷贝](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100910.png)

## 总结

写了这么多主要还是了解一些对象的拷贝问题，从上面的一步步改造也可以看出来要真想写完美这个功能也是得一番功夫的。所以最后大家还是去用 lodash 吧，哈哈哈哈哈。
