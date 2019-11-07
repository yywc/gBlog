## 前言

平时的开发中可能不太需要用到设计模式，但是 JS 用上设计模式对于性能优化和项目工程化也是很有帮助的，下面就对常用的设计模式进行简单的介绍与总结。

## 1. 单例模式

定义：保证一个类仅有一个实例，并提供一个访问它的全局访问点。
```js
class Singleton {
  constructor(age) {
    this.age = age;
  }
  static getInstance(age) {
    const instance = Symbol.for('Singleton'); // 隐藏属性，伪私有
    if (!Singleton[instance]) {
      Singleton[instance] = new Singleton(age);
    }
    return Singleton[instance];
  }
}

const singleton = Singleton.getInstance(30);
const singleton2 = Singleton.getInstance(20);
console.log(singleton === singleton2); // true
```

## 2. 策略模式

定义：定义一系列的算法，把它们一个个封装起来，并且使它们可以相互替换。



策略模式的核心是整个分为两个部分：

+ 第一部分是策略类，封装具体的算法；

+ 第二部分是环境类，负责接收客户的请求并派发到策略类。



现在我们假定有这样一个需求，需要对表现为 S、A、B 的同事进行年终奖的计算，分别对应为 4 倍、3 倍、2 倍工资，常见的写法如下:

```javascript
const calculateBonus = (performanceLevel, salary) => {

  if (performanceLevel === 'S') {
    return salary * 4;
  }

  if (performanceLevel === 'A') {
    return salary * 3;
  }

  if (performanceLevel === 'B') {
    return salary * 2;
  }

};

calculateBonus('B', 20000); // 40000
```

可以看到，代码里面有较多的 if else 判断语句，如果对应计算方式改变或者新增等级，我们都需要对函数内部进行调整，且薪资算法重用性差，于是我们可以通过策略模式来进行重构，代码如下：

```javascript
// 解决魔术字符串
const strategyTypes = {
  S: Symbol('S'),
  A: Symbol('A'),
  B: Symbol('B'),
};

// 策略类
const strategies = {
  // S 级工资计算
  [strategyTypes.S](salary) {
    return salary * 4;
  },

  // A 工资计算
  [strategyTypes.A](salary) {
    return salary * 3;
  },

  // B 工资计算
  [strategyTypes.B](salary) {
    return salary * 2;
  }
  // 更多级别计算可以自由添加，且不会对原有部分造成影响
};

// 环境类
const calculateBonus = (level, salary) => { 
  return strategies[level](salary);
};

calculateBonus(strategyTypes.S, 300); // 1200
```

策略模式的优点：

+ 利用组合、委托、多态等技术和思想，有效地避免了多重 if-else 语句；

+ 提供了对开放-封闭原则的完美支持，将算法封装在独立的 strategy 中，使得它们易于切换、理解、扩展；

+ strategy 中的算法也可以用在别处，避免许多复制粘贴；



缺点：

+ 增加许多策略类或策略对象；

+ 违反知识最少原则；

## 3. 代理模式

定义：为一个对象提供一个代用品或占位符，以便控制对它的访问。



### 3.1 虚拟代理
在程序世界里，操作可能是昂贵的，这时候 B 通过监听 C 的状态来将 A 的请求发送过去（原本 A 需要实时去访问 C准备请求），减少开销。

**代理的意义**
> 单一职责: 就一个类(通常也包括对象和函数等)而言，应该仅有一个引起它变化的原因。如果一个对象承担了多项职责，就意味着这个对象将变得巨大，引起它变化的原因可能会有多个。

例子：图片加载前先显示一张 loading 图（预加载）。
```javascript
// 立即执行函数创建 image，闭包设置 src
const myImage = (() => {
    const imgNode = document.createElement('img'); 
    document.body.appendChild(imgNode);
    return (src) => {
      imgNode.src = src;
    }
})();

// 代理 myImage，在 test.jpg onload 之前显示 loading.gif
const proxyImage = (() => {
    const img = new Image();
    img.onload = () => {
      myImage(this.src);
    }
    return (src) => {
      myImage('./loading.gif');
      img.src = src;
    }
})();

proxyImage('./test.jpg');
```
这里的 myImage 只进行图片 src 的设置，其他代理的工作交给了 proxyImage 方法，符合单一职责原则。此外，也保证了代理和本体接口的一致性。

### 3.2 缓存代理

缓存代理可以为一些开销大的运算结果提供暂时的存储，在下次运算时，如果传递进来的参数跟之前一致，则可以直接返回前面存储的运算结果。

例子：计算乘积，缓存 ajax 数据。
```javascript
// 计算所有参数的乘积
const multi = (...args) => {
  let result = 1;
  args.forEach(arg => {
    result = result * arg;
  });
  return result;
};

// 缓存计算结果函数
const proxyMulti = (() => {
  const cache = {}; // 缓存池
  return (...args) => {
    const param = args.join(',');
    if (param in cache) {
      return cache[param]; // key 为 1,2,3,4，值为 24 的对象
    }
    cache[param] = multi.apply(this, args);
    return cache[param];
  };
})();

console.time();
console.log(proxyMulti(1, 2, 3, 4)); // 24
console.timeEnd(); // 约 0.7 ms

console.time();
console.log(proxyMulti(1, 2, 3, 4)); // 24
console.timeEnd(); // 约 0.1ms
```

## 4. 观察者模式

观察者模式又叫发布—订阅模式，它定义对象间的一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都将得到通知。在 JavaScript 开发中，我们一般用事件模型来替代传统的观察者模式。

### 4.1 DOM 事件

最早接触到的观察者模式大概就是 DOM 事件了，比如用户的点击操作。我们没办法知道用户什么时候点击，但是当用户点击时，被点击的节点就会向订阅者发布消息。

```javascript
document.body.addEventListener('click', () => {
  alert('我被点击啦!');
});
```

### 4.2 自定义事件
要实现自定义事件，需要进行三步：
1. 指定发布者；
2. 给发布者添加一个缓存列表，用以通知订阅者；
3. 遍历缓存列表依次触发存放在里面的订阅者的回调函数；
```javascript
class Event {
  constructor() {
    this.eventListObj = {}; // 事件列表对象
  }

  // 单例
  static getInstance() {
    const instance = Symbol.for('instance');
    if (!Event[instance]) {
      Event[instance] = new Event();
    }
    return Event[instance];
  }

  // 添加监听事件，同一指令，可以有多个事件
  listen(key, fn) {
    if (!this.eventListObj[key]) {
      this.eventListObj[key] = [];
    }
    // 订阅消息添加进缓存列表
    this.eventListObj[key].push(fn);
  }

  // 触发监听事件
  trigger(key, ...args) {
    const fns = this.eventListObj[key];
    if (fns && fns.length !== 0) {
      fns.forEach(fn => {
        fn.apply(this, args);
      });
    }
  }

  // 移除监听事件
  remove(key, fn) {
    let fns = this.eventListObj[key];
    // 被订阅过才操作
    if (fns) {
      // 根据 fn 参数来判断是全部移除还是指定移除
      if (!fn) {
        fns.length = 0; // 移除全部
      } else {
        // 移除某一个
        fn.forEach((f, index) => {
          if (f === fn) {
            fns.splice(index, 1);
          }
        });
      }
    }
  }
}

const event = Event.getInstance(); // 创建全局发布者

const add = (a, b) => {
  console.log(a + b);
};
const minus = (a, b) => {
  console.log(a - b);
};

event.listen('add', add); // 订阅加法消息
event.listen('minus', minus); // 订阅减法消息

event.trigger('add', 1, 3); // 触发加法订阅消息
event.trigger('minus', 3, 1); // 触发减法订阅消息

console.log(event); // Event 对象 eventListObj 属性包含 add

event.remove('add', add); // 取消加法订阅事件
console.log(event); // Event 对象 eventListObj 属性不包含 add
```
执行结果：

例子：ajax 请求登录后进行多种操作，以及在 vue 中 emit 和 on，node.js 中的 events

## 5. 模板方法模式

模板方法模式是一种只需使用继承就可以实现的非常简单的模式。

模板方法模式由两部分结构组成，第一部分是抽象父类，第二部分是具体的实现子类。

通常在抽象父类中封装了子类的算法框架，包括实现一些公共方法以及封装子类中所有方法的执行顺序。

子类通过继承这个抽象类，也继承了整个算法结构，并且可以选择重写父类的方法。

下面我们来举个例子——假如我们要泡一杯茶和一杯咖啡步骤如下：
1. 把水煮沸
2. 用沸水 ( 冲泡咖啡 / 浸泡茶叶 )
3. 把 ( 咖啡 / 茶水 ) 倒进杯子
4. 加糖和牛奶 / 加柠檬

很容易发现其中第一步是共有的，其他步骤大体一致，那么我们就可以使用模板方法来实现它。( 假如有人不想加糖和牛奶怎么办呢？ )

```javascript
// 抽象出饮料类用来表示咖啡和茶
class Beverage {
  // 钩子：解决了有人不想加糖和牛奶的问题
  customerWantsCondiments = true;
  init() {
    this.boilWater();
    this.brew();
    this.pourInCup();
    if (this.customerWantsCondiments) {
      this.addCondiments();
    }
  }
  // 第一步：把水煮沸
  boilWater(){
    console.log('把水煮沸');
  }
  // 第二步：冲泡饮料，在子类中重写
  brew(){
    throw new Error('brew function must override in child');
  }
  // 第三步：倒出饮料，在子类中重写
  pourInCup(){
    throw new Error('pourInCup function must override in child');
  }
  // 第四步：个性化饮料，在子类中重写
  addCondiments(){
    throw new Error('addCondiments function must override in child');
  }
}

class Coffee extends Beverage {
  customerWantsCondiments = false;
  brew(){
    console.log('用沸水冲泡咖啡');
  }
  pourInCup(){
    console.log('把咖啡倒进杯子');
  }
  addCondiments(){
    console.log('加糖和牛奶');
  }
}

class Tea extends Beverage {
  brew(){
    console.log('用沸水浸泡茶叶');
  }
  pourInCup(){
    console.log('把茶水倒进杯子');
  }
  addCondiments(){
    console.log('加柠檬');
  }
}

new Coffee().init(); // 把水煮沸、用沸水冲泡咖啡、把咖啡倒进杯子
console.log('---------------');
new Tea().init(); // 把水煮沸、用沸水浸泡茶叶、把茶水倒进杯子、加柠檬
```

## 6. 职责链模式

职责连模式：通过把对象连成一条链，让请求沿着这条链传递，直到有一个对象能处理为止，解决了发送者和接收者之间的耦合。

> A --> B --> C --> ... --> N，中间有一个对象能处理 A 对象的请求，如果没有需要在最后处理异常。

现实中的例子：早高峰挤公交的时候递公交卡，只需要往前递，总会递到售票员手里刷卡，而不用管递给了谁，下面举一个实际的例子来看看。

假如现在有个电商定金优惠券功能

+ 付 500 元定金可以获得 100 元优惠券且一定能买到商品；
+ 付 200 元定金可以获得 50 元优惠券且一定能买到商品；
+ 如果付定金只能进入普通购买，需要在库存足够的时候才可以买到商品；
+ 不付定金就是普通购买；


我们定义一个函数，接收三个参数：

+ orderType：1、2、3 分表代表 500 元定金， 200 元定金和无定金模式；
+ pay：true、false 代表拍下订单是否付款；
+ stock：number 代表库存余量；

```javascript
const order = (orderType, pay, stock) => {
  if (orderType === 1) {
    if (pay === true) {
      console.log('获得 100 元优惠券');
    } else {
      if (stock > 0) {
        console.log('普通购买, 无优惠券');
      } else {
        console.log('库存不足');
      }
    }
  } else if (orderType === 2) {
    if (pay === true) {
      console.log('获得 50 元优惠券');
    } else {
      if (stock > 0) {
        console.log('普通购买, 无优惠券');
      } else {
        console.log('库存不足');
      }
    }
  } else if (orderType === 3) {
    if (stock > 0) {
      console.log('普通购买, 无优惠券');
    } else {
      console.log('库存不足');
    }
  }
}
order(1, true, 20); // 获得 100 元优惠券
```
这显然不是一段好代码，大量的 if else 条件分支，如果业务再复杂一点，最后根本就没法看了。

那么我们通过 AOP(面向切面编程) 实现职责链：

```javascript
const order500 = (orderType, pay, stock) => {
  // 如果支付了 500 元定金，就成功获得 100 元优惠券
  if (orderType === 1 && pay === true) {
    return console.log('已支付定金，获得100元优惠券');
  }
  // 否则进入下一步
  return 'NEXT';
}

const order200 = (orderType, pay, stock) => {
  // 如果支付了 200 元定金，就成功获得 50 元优惠券
  if (orderType === 2 && pay === true) {
    return console.log('已支付定金，获得50元优惠券');
  }
  // 否则进入下一步
  return 'NEXT';
}

// 普通购买模式
const orderNormal = (orderType, pay, stock) => {
  // 如果库存大于 0，可以购买
  if (stock > 0) {
    return console.log('普通购买，无优惠券');
  }
  // 否则库存不足，无法购买
  return console.log('库存不足');

}

// 给 Funciton 挂载 after 方法，通过 NEXT 判断是否进行下一步
Function.prototype.after = function(fn) {
  const self = this;
  return (...args) => {
    const result = self.apply(this, args);
    return result === 'NEXT' ? fn.apply(this, args) : result;
  }
}

const order = order500.after(order200).after(orderNormal); // 获得 order 计算方式
order(1, false, 10); // 普通购买，无优惠券
```

通过分解成三个独立的函数，返回处理不了的结果'NEXT'，交给下一个节点处理。通过 after 来进行绑定，最后我们在新增需求的时候可以在 after 中间插入即可，耦合度大大降低，但是这样也有一个不好的地方，职责链过长增加了函数的作用域。

## 7. 中介者模式

在程序里，对象经常会和其他对象进行通信，当项目比较大，对象很多的时候，这种通信就会形成一个通信网，当我们想要修改某一个对象时，需要十分小心，以免这些改动牵一发而动全身，导致出现BUG，非常的复杂。

中介者模式就是用来解除这些对象间的耦合，形成简单的对象到中介者到对象的操作。

下面以现实中的机场指挥塔为例说明。

+ 如果没有指挥塔的情况，每一架飞机都需要和其他飞机进行通信，确保航线的安全，我们假设目的地相同就为航线不安全：

```javascript
// 飞机类
class Plane {
  constructor(name, to) {
    this.name = name;
    this.to = to;
    this.otherPlanes = []; // 其他飞机集合
  }
  success() {
    console.log(`${this.name} 可以正常飞行`);
  }
  fail(plane) {
    console.log(`${this.name} 与 ${plane.name} 航线冲突，请调整`);
  }
  fly() {
    let normal = true; // 标志位，是否可以正常飞行
    let targetPlane = {};
    for (let i = 0; i < this.otherPlanes.length; i++) {
      // 需要与其他每一架飞机比较，如果其他飞机与当前飞行航线冲突，则不可以飞行，
      if (this.otherPlanes[i].to === this.to) {
        normal = false;
        targetPlane = this.otherPlanes[i]; // 记住与当前飞机冲突的飞机
        break;
      }
    }
    if (normal) {
      this.success(); // 成功，可以飞行
      return;
    }
    this.fail(targetPlane); // 失败，报告冲突飞机
  }
}

// 飞机工厂，创建飞机对象
class PlaneFactory {
  constructor() {
    this.planes = []; // 所有创建出来的飞机集合
  }
  static getInstance() {
    const instance = Symbol.for('instance');
    if (!PlaneFactory[instance]) {
      PlaneFactory[instance] = new PlaneFactory();
    }
    return PlaneFactory[instance];
  }
  // 创建飞机，飞机名称和目的地
  createPlane(name, to) {
    const plane = new Plane(name, to);
    this.planes.push(plane);
    this.planes.forEach(planeItem => {
      // 如果飞机名称与其他飞机名称不一样，将其他飞机放入 otherPlanes 数组，用以目的地比较
      if (plane.name !== planeItem.name) {
        plane.otherPlanes.push(planeItem);
      }
    });
    // 返回当前飞机
    return plane;
  }
}

// 获取飞机工厂实例
const planeFactory = PlaneFactory.getInstance();

// 创建四架飞机
const planeA = planeFactory.createPlane('planeA', 1);
const planeB = planeFactory.createPlane('planeB', 2);
const planeC = planeFactory.createPlane('planeC', 3);
const planeD = planeFactory.createPlane('planeD', 2);

planeA.fly(); // planeA 可以正常飞行
planeB.fly(); // planeB 可以正常飞行
planeC.fly(); // planeC 可以正常飞行
planeD.fly(); // planeD 与 planeB 航线冲突，请调整
```

当飞机足够多时，这样的方式就会变得非常复杂，而且某一天有飞机出故障维修不参与飞行，那么改动也是麻烦的。

+ 存在指挥塔的情况，飞机不需要知道其他飞机的存在，只需要向指挥塔通信即可，而且添加了移除故障飞机的方法。

```javascript
/**
  * 指挥塔模式：中介者
  * 指挥塔：接收飞机传递过来的一切信息，同时操作需要操作的飞机
  * 飞机类：定义飞机对象，具备名称和目的地，同时具备向指挥塔通信的方法
  * 飞机工厂类：定义创建飞机对象的工厂，在创建时通知指挥塔添加飞机
  */
// 指挥塔
class Tower {
  constructor() {
    this.planes = []; // 飞机集合
    // 操作飞机对象
    this.operations = {
      add: this.add,
      remove: this.remove,
      fly: this.fly,
    };
  }
  static getInstance() {
    const instance = Symbol.for('instance'); // 防止被覆盖
    if (!Tower[instance]) {
      Tower[instance] = new Tower();
    }
    return Tower[instance];
  }
  // 接收飞机传递给指挥塔的信息
  receiveMessage(msg, ...args) {
    this.operations[msg].apply(this, args);
  }
  // 添加飞机
  add(plane) {
    this.planes.push(plane);
  }
  // 移除飞机
  remove(plane) {
    for (let i = 0; i < this.planes.length; i++) {
      if (this.planes[i].name === plane.name) {
        this.planes.splice(i, 1);
      }
    }
  }
  // 飞机开始飞行
  fly(plane) {
    let normal = true;
    let targetPlane = {};
    for (let i = 0; i < this.planes.length; i++) {
      // 如果当前飞机与所有飞机名称不一致，但是航线一致，则认为航线冲突
      // 原本这个步骤放在了每个飞机里，现在由指挥塔进行
      if (
        this.planes[i].name !== plane.name &&
        this.planes[i].to === plane.to
      ) {
        normal = false;
        targetPlane = this.planes[i];
        break;
      }
    }
    if (normal) {
      plane.success();
      return;
    }
    plane.fail(targetPlane);
  }
}

// 获得指挥塔实例
const tower = Tower.getInstance();

// 飞机类：只需要设置名称和向指挥塔通信的方法就可以
class Plane {
  constructor(name, to) {
    this.name = name;
    this.to = to;
  }
  success() {
    console.log(`${this.name} 可以正常飞行`);
  }
  fail(plane) {
    console.log(`${this.name} 与 ${plane.name} 航线冲突，请调整`);
  }
  // 通知指挥塔该飞机移除
  remove() {
    tower.receiveMessage('remove', this);
  }
  // 通知指挥塔该飞机开始飞行
  fly() {
    tower.receiveMessage('fly', this);
  }
}

// 飞机工厂：创建飞机，同时向指挥塔通知 add 方法，添加飞机
class PlaneFactory {
  static plane(name, to) {
    const plane = new Plane(name, to);
    tower.receiveMessage('add', plane);
    return plane;
  }
}

// 创建四架飞机
const planeA = PlaneFactory.plane('planeA', 1);
const planeB = PlaneFactory.plane('planeB', 2);
const planeC = PlaneFactory.plane('planeC', 3);
const planeD = PlaneFactory.plane('planeD', 2);

planeA.fly(); // planeA 可以正常飞行
planeB.fly(); // planeB 与 planeD 航线冲突，请调整
planeC.fly(); // planeC 可以正常飞行
planeD.fly(); // planeD 与 planeB 航线冲突，请调整
planeD.remove(); // 假如 planeD 出故障了，进行移除
planeB.fly(); // planeB 可以正常飞行
```

中介者模式是知识最少原则的一种实现，是指一个对象尽可能少的了解其他的对象，如果对象之间的耦合度过高，一个对象发生改变之后，难免会影响到其他对象，在中介者模式中，对象几乎不知道其他对象的存在，它们只能通过中介者对象来通信。但是这样的结果就是中介者对象难免会变的臃肿。

## 8. 装饰者模式

装饰者（decorator）模式：给对象动态地增加职责的方式。

我们在开发中经常会使用到，因为在 JavaScript 中对对象动态操作是一件再简单不过的事情了。

```javascript
const person = {
  name: 'shelly',
  age: 18,
}
person.job = 'student';
```

### 8.1  装饰函数

给对象扩展属性和方法相对简单，但是在改写函数时却不是那么容易，尤其是尽量保证开放-封闭原则的前提下。我们可以通过使用 AOP 装饰函数来达到理想的效果。

```javascript
let add = (a, b) => {
  console.log(a + b);
};
// 在函数执行之前执行
Function.prototype.before = function(beforeFn) {
  const self = this;
  return (...args) => {
    beforeFn.apply(this, args);
    return self.apply(this, args);
  };
};
// 在函数执行之后执行
Function.prototype.after = function(afterFn) {
  const self = this;
  return (...args) => {
    const result = self.apply(this, args);
    afterFn.apply(this, args);
    return result;
  };
};
// 装饰 add 函数
add = add
  .before(() => {
    console.log('before add');
  })
  .after(() => {
    console.log('after add');
  });

add(1, 2); // before add、3、after add
```

## 9. 设计原则和编程技巧

### 9.1 单一职责原则

单一职责原则（SRP）：一个对象（方法）只做一件事情。如果一个方法承担了过多的职责，将来改写它的可能性就越大。

这一原则在单例模式、代理模式中都有广泛的应用。

**何时该分离**？

这是很难把控的一个点，比如 ajax 请求，创建 xhr 对象和发送请求虽然是两个职责，但是他们是一起变化，可以不用分离；像 jQuery 的 attr 方法，既赋值，又取值，理论上应该分离，却方便了用户。所以需要我们在实际上拿捏。

### 9.2 最少知识原则

最少知识原则（LKP）：一个软件实体应当尽可能地少于其他实体发生相互作用。这里的实体包括了对象、类、模块、函数等。

常见的做法是引入第三方对象来承担多个对象间的通信，例如中介者模式、封装。

### 9.3 开放 - 封闭原则

开放 - 封闭原则（OCP）：软件实体（类、模块、函数）等应该是可以扩展的，但是不可修改。

OCP 在几乎所有的设计模式中得到了很好的表现。

#### 9.3.1 扩展

假如我们要修改一个函数，业务逻辑极其复杂，那么我们遵守开放 - 封闭原则在原来的基础绑定一个 after 方法，传入回调函数实现我们新的需求而不用去改变之前的代码。

```javascript
// 原来的函数
let theMostComplicatedFn = (a, b) => {
  console.log('我是极其复杂的函数');
  console.log(a + b);
};
// 定义在原来函数执行后执行的函数
theMostComplicatedFn.after = function(afterFn) { // 这里由于 this 指向问题，需要用 fucntion
  const self = this;
  return (...args) => {
    const result = self.apply(this, args);
    afterFn.apply(this, args);
    return result;
  };
};
// 混合后的函数
theMostComplicatedFn = theMostComplicatedFn.after((a, b) => {
  console.log(a, b);
});

theMostComplicatedFn(1, 2); // 我是极其复杂的函数、3、1 2
```

#### 9.3.2 多态

利用对象的多态性也可以让程序遵循开放 - 封闭原则，这是一个常用的技巧。

我们都知道猫吃鱼，狗吃肉，那么我们用代码来表达一下。

```javascript
const food = (animal) => {
  if (animal instanceof Cat) {
    console.log('猫吃鱼');
  } else if (animal instanceof Dog) {
    console.log('狗吃肉');
  }
}

class Dog {}
class Cat {}
food(new Dog()); // 狗吃肉
food(new Cat()); // 猫吃鱼
```

有一天加入了羊，又得再加一个 else if 来判断，如果很多呢？那么我们就要一直去改变 food 函数，这显然不是一种好的方法。我们现在可以利用多态性，将共同的 food 抽取出来。

```javascript
const food = (animal) => {
  animal.food();
}

class Dog {
  food() {
    console.log('狗吃肉');
  }
}
class Cat {
  food() {
    console.log('猫吃鱼');
  }
}
class Sheep {
  food() {
    console.log('羊吃草');
  }
}
food(new Dog()); // 狗吃肉
food(new Cat()); // 猫吃鱼
food(new Sheep()); // 羊吃草
```

这样，当我们以后要增加新的动物时，就不需要每次都去改变 food 函数了。

#### 9.3.3 其他方式

钩子函数、回调函数。

### 9.4 代码重构

#### 9.4.1 提炼函数

把一段代码提炼成函数的好处是：
+ 避免出现超大函数
+ 独立出来的函数有利于代码复用
+ 独立出来的函数更容易被覆写
+ 独立出来的函数如果有一个好的命名，它本身就起到了注释的作用

#### 9.4.2 合并重复的条件片段

如果一个函数体内有一些条件分支语句，而这些条件分支语句的内部散布了一些重复的代码，那么就有必要进行合并去重工作。

#### 9.4.3 把条件分支语句提炼成函数

下面是一个例子：

```javascript
const getPrice = (price) => {
  const date = new Date();
  if (date.getMonth() >= 6 && date.getMonth() <=9) { // 夏天
    return price * 0.8;
  }
  return price;
}
```

条件语句乍一看需要理解一会儿，那么此处可以做一下调整：

```javascript
// 通过函数名也起到了注释作用
const isSummer = () => {
  const month = new Date().getMonth();
  return month >= 6 && month <=9;
}

const getPrice = function (price) {
  const date = new Date();
  if (isSummer()) {
    return price * 0.8;
  }
  return price;
}
```

#### 9.4.4 合理使用循环

在函数体内，如果有些代码实际上负责的是一些重复性的工作，那么合理利用循环不仅可以完成同样的功能，还可以使代码量更少。我们以创建 xhr 对象为例：

```javascript
const createXHR = () => {
  let xhr;
  try {
    xhr = new ActiveXObject('MSXML2.XMLHttp.6.0');
  } catch (e) {
    try {
      xhr = new ActiveXObject('MSXML2.XMLHttp.3.0');
    } catch (e) {
      xhr = new ActiveXObject('MSXML2.XMLHttp');
    }
  }
  return xhr;
};
const xhr = createXHR();
```

下面我们通过循环，可以达到和上面一样的效果：

```javascript
const createXHR = function () {
  const versions = ['MSXML2.XMLHttp.6.0ddd', 'MSXML2.XMLHttp.3.0', 'MSXML2.XMLHttp'];
  for (let i = 0, version; version = versions[i++];) {
    try {
      return new ActiveXObject(version);
    } catch (e) {
    }
  }
};
const xhr = createXHR();
```

#### 9.4.5 提前让函数退出代替嵌套条件分支

在多层条件分支语句中，我们可以挑选一些分支，在进入这些分支后，就立即让函数退出，减少非关键代码的混淆。

#### 9.4.6 传递对象参数代替过长的参数列表

函数参数过长过多会引起调用调用者的不适，可能出现传少或传反的情况。如果有这种情况，我们可以通过将参数包装成一个对象传入函数，然后在函数体内进行取值就可以了。

## 总结

在平时开发的过程中，策略模式、职责链模式还有装饰者模式可能用的情况比较多，中介者和观察者经常可以看到，虽然自己不一定亲自设计，但是从原理上理解了他们也能方便我们更好地使用。