## 前言

在使用 Vue 开发的过程中，一旦项目到达一定的体量或者模块化粒度小一点，就会遇到兄弟组件传参，全局数据缓存等等问题。这些问题也有其他方法可以解决，但是通过 `Vuex` 来管理这些数据是最好不过的了。

该系列分为上中下三篇：

+ [Vuex 从使用到原理分析（上篇）]()：介绍 Vuex，以及 Vuex 的几种常见写法；
+ [Vuex 从使用到原理分析（中篇）]()：分析 Vuex 的初始化以及模块获取安装；
+ [Vuex 从使用到原理分析（下篇）]()：分析 Vuex 和 Store 中的一些方法包括辅助函数；

## 1. 什么是 Vuex ？

Vuex 是一种**状态管理模式**，它集中管理应用内的所有组件状态，并在变化时可追踪、可预测。

也可以理解成一个数据仓库，仓库里数据的变动都按照某种严格的规则。



国际惯例，上张看不懂的图。慢慢看完下面的内容，相信这张图不再那么难以理解。

![vuex](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-09-20-063358.jpg)

### 状态管理模式

在 vue 单文件组件内，通常我们会定义 `data`、`template`、`methods` 等。

+ `data` 通常作为数据源保存我们定义的数据

+ `template` 作为视图映射我们 data 中的数据

+ `methods` 则会响应我们一些操作来改变数据

这样就构成了一个简单的状态管理。

## 2. 为什么要使用 Vuex？

很多情况下，我们使用上面举例的状态自管理应用也能满足场景了。但是如前言里所说，全局数据缓存（例如省市区的数据），兄弟组件数据响应（例如单页下 `Side` 组件和 `Header` 组件参数传递）就会破坏单向数据流，而破坏的代价是很大的，轻则“卧槽，这是谁写的不可回收垃圾，噢，是我！”，重则都无法理清逻辑来重构。

而 `Vuex` 的出现则解决了这一难题，我们不需要知道数据具体在哪使用，只需要去通知数据改变，然后在需要使用到的地方去使用就可以了。

## 3. 需不需要使用 Vuex？

首先要确定自己的需求是不是有那么大...

如果确定数据寥寥无几，那使用一个 store 模式来管理就可以了，杀鸡不用宰牛刀。

下面用一个全局计数器来举例。

`store.js`&`main.js`：

```js
// store.js
export default {
  state: {
    count: 0,
  },
  // 计数增加
  increaseCount() {
    this.state.count += 1;
  },

  // 计数归零
  resetCount() {
    this.state.count = 0;
  },
};

// main.js
import store form './store';
Vue.prototype.$store = store;
```

`App.vue`：

```vue
<template>
  <div id="app">{{ state.count }}</div>
</template>

<script>
export default {
  name: 'App',
  data() {
    return {
      state: this.$store.state,
    };
  },
  mounted() {
    // 2 秒后计数加 1，视图会变化
    setTimeout(() => {
      this.$store.increaseCount();
    }, 2000);
  },
};
</script>
```

像这样就完成了一个简单的全局状态管理，但是，这样 state 中的数据不是响应式的，这里是通过绑定到了 data 下的 state 中达到响应的目的，当我们需要用到共享的数据是实时响应且能引发视图更新的时候该如何做呢？

## 4. 如何使用 Vuex？

既然知道了自己要使用 `Vuex`，那么如何正确地使用也是一门学问。

### 4.1 Vuex 核心概念简介

`Vuex `的核心无外 `State`、`Getter`、`Mutation`、`Action`、`Module` 五个，下面一个个来介绍他们的作用和编写方式。

#### State

`Vuex`的唯一数据源，我们想使用的数据都定义在此处，唯一数据源确保我们的数据按照我们想要的方式去变动，可以通过 store.state 来取得内部数据。

#### Getter

store 的计算属性，当我们需要对 state 的数据进行一些处理的时候，可以先在 getters 里进行操作，处理完的数据可以通过 store.getters 来获取。

#### Mutation

`Vuex`让我们放心使用的地方也就是在这，store 里的数据只允许通过 mutation 来修改，避免我们在使用过程中覆盖 state 造成数据丢失。

如果我们直接通过 store.state 来修改数据，vue 会抛出警告，并无法触发 mutation 的修改。

![image-20190911172051077](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-09-20-063359.jpg)

每一个 mutation 都有一个 type 和 callback 与之对应，可以通过 store.commit(type[, payload]) 来提交一个 mutation 到 store，修改我们的数据。

ps: payload 为额外参数，可以用来修改 state。

#### Action

在 Mutation 中并不允许使用异步操作，当我们有异步操作（例如 http 请求）时，就必须在 Action 中来完成了。

Action 中提交的是 mutation，不是直接变更数据，所以也是允许的操作。

我们可以通过 store.dispatch(action) 来触发事件，进行相关操作后，通过 commit 方法来提交 mutation。

#### Module

`Vuex` 的管理已经很美好了，但是！全局数据还好，反正很多地方会要用到，如果只是某个单页下的兄弟组件共享某些数据呢，那这样大张旗鼓地放到`Vuex`中，久而久之便会臃肿无法管理，故 Module 就是解决这个问题。

每个 Module 就是一个小型的 store，与上面几个概念如出一辙，唯一不同的是 Module 支持命名空间来更好的管理模块化后的 store。

**强烈建议每一个模块都写上。**

> namespaced: true

设置这个属性后，我们就需要通过 store.state.moduleName 获取 Module 下的 state 里的数据，而 commit 和 dispatch 需要在传递 type 的时候加入路径（也就是模块名）store.commit('moduleName/type', payload)。

### 4.2 入门版食用方式

我们以下所说都是以 vue-cli 脚手架生成的项目为例来介绍几种常规写法。

当我们的需要共享的数据量很小时，只需要简单的写在 store.js 文件里就可以了，而且不需要使用到 Module，使用方式也比较简单。

`store.js`：

```js
// store.js
import Vue from 'vue';
import Vuex from 'vuex';

Vue.use(Vuex);
const vm = new Vue();

export default new Vuex.Store({
  state: {
    count: 0,
  },
  mutations: {
    setCount(state, count) {
      state.count = count;
    },
  },
  getters: {
    getCountPlusOne(state) {
      return state.count + 1;
    },
  },
  actions: {
    setCountAsync({ commit }, count) {
      return new Promise(async resolve => {
        const { data: count } = await vm.$http('/api/example', { count });
        commit('setCount', count);
        resolve(count);
      });
    },
  },
});
```

`main.js`：

```js
// main.js
import Vue from 'vue';
import App from './App';
import store from './store';

Vue.config.productionTip = false;

/* eslint-disable no-new */
new Vue({
  el: '#app',
  store,
  template: '<App />',
  components: { App },
});
```

那么我们在组件中需要用到时就可以通过以下操作获得想要的结果了：

+ **this.$store.state.count**：实时获取 count 的值
+ **this.$store.getters.getCountPlusOne**：获取 count + 1 的值并缓存
+ **this.$store.getters.getCountPlusOne**()：每次都会计算一个新的结果
+ **this.$store.commit('setCount', 5)**：实时改变 count 的值
+ **this.$store.dispatch('setCountAsync', 3)**：派发事件，异步更新 count 的值

在 commit 和 dispatch 的时候，如果需要传入多个参数，可以使用对象的方式。

e.g.

```js
this.$store.commit('setCount', { 
  count: 5,
  other: 3 
}); 
// 或者如下
thit.$store.commit({ 
  type: 'setCount',
  count: 5,
  other: 3,
});
```

### 4.3 进阶版食用方式

一般来说，入门版食用方式真的不太推荐，因为项目写着写着就和你的身材一样，一天天不忍直视。故需要进阶版来优化优化我们的“数据仓库”了。

总的来说，核心概念还是那样，只是我们将 store 按照核心概念进行拆分，并将一些常数固定起来，避免拼写错误这种弱智出现，也方便哪天需要修改。

整个项目结构如下：

store/
├── actions.js 
├── getters.js
├── index.js
├── mutation-types.js
├── mutations.js
└── state.js

我们依旧以上面的例子为例来改写：

#### mutation-types.js

这个文件也是强烈建议编写的，将 mutation 的方法名以常量的方式定义在此处，在其他地方通过 `import * as types from './mutation-types';` 来使用，第一可以避免手抖拼写错误，第二可以方便哪天需要改动变量名，改动一处即可。

```js
export const SET_COUNT = 'SET_COUNT';
```

#### state.js

```js
export default {
  count: 0,
}
```

#### getters.js

```js
getCountPlusOne(state) {
  return state.count + 1;
};
```

####  actions.js

```js
import Vue from 'vue';
import * as types from './mutation-types';

const vm = new Vue();

export const setCountAsync = ({ commit }, count) => {
  return new Promise(async resolve => {
    const { data: count } = await vm.$http('/api/example', { count });
    commit('setCount', count);
    resolve(count);
  });
};
```

#### mutations.js

```js
import * as types from './mutation-types';

export default {
  [types.SET_COUNT](state, count) {
    state.count = count;
  },
};
```

#### index.js

```js
import Vue from 'vue';
import Vuex from 'vuex';
import createLogger from 'vuex/dist/logger';
import * as actions from './actions';
import * as getters from './getters';
import state from './state';
import mutations from './mutations';
import * as types from './mutation-types';

Vue.use(Vuex);

const debug = process.env.NODE_ENV !== 'production';

const logger = createLogger(); // 引入日志，帮助我们更好地追踪 mutaion 触发的 state 变化

export default new Vuex.Store({
  actions,
  getters,
  state,
  mutations,
  strict: debug,
  plugins: debug ? [logger] : [],
});
```

在项目中我们可以通过 this.$store 来获取数据或者提交 mutation 等。同时也可以使用辅助函数来帮助我们更加便捷的操作 store，这部分内容放到[高级版食用方式](#4.4 高级版食用方式)里介绍。

至此，我们编写了一个完成进阶版的食用方式，大多数项目通过这样的结构来管理 store 也不会让一个 store.js 文件成百上千行了。

但是，又是但是。我有些数据仅仅在小范围内使用，写在这个里面，体量一多不还是会找不到北吗？

那请继续看下面的高级版食用方式。

### 4.4 高级版食用方式

所谓的高级版食用方式也就是在进阶版的基础上将大大小小的数据分解开来，让他们找到自己的归宿。

关键的概念就是 `Module`了，通过这个属性，我们可以将数据仓库拆解成一个个小的数据仓库来管理，从而解决数据冗杂问题。

这里的写法有两种，一种是将 module 写到 store 中进行统一管理；另一种则是写在模块处，我个人喜欢第二种，就近原则。

```js
// store/
// + └── modules.js 

// src/helloWorld/store.js
export const types = {
  SET_COUNT: 'SET_COUNT',
};

export const namespace = 'helloWorld';

export default {
  name: namespace, // 这个属性不属于 Vuex，但是通过常量定义成模块名，避免文件间耦合字符串
  namespaced: true, // 开启命名空间，方便判断数据属于哪个模块
  state: {
    count: 0,
  },
  mutations: {
    [types.SET_COUNT](state, count) {
      state.count = count;
    },
  },
  getters: {
    getCountPlusOne(state) {
      return state.count + 1;
    },
  },
};


// store/modules.js
import helloWorld from '@/views/helloWorld/store';
export default {
  [helloWorld.name]: helloWorld,
}

// store/index.js
+ import modules from './modules';
export default new Vuex.Store({
+  modules,
});
```

由于 module 里的数据不多，所以我们写在一个 store 文件内更为方便，当然了，如果你的数据够多，这里继续拆分也是可以的。 高级版食用方式一般这样也差不多了，下面补充几个注意点。

#### module store 的操作方式

与全局模式差不多，只多了在获取使用到命名空间。

+ 提交 Mutation：this.\$store.commit('helloWorld/SET_COUNT', 1);
+ 提交 Action：this.\$store.dispatch('helloWorld/SET_COUNT', 2);
+ 获取 Gtters：this.\$store.getters['helloWorld/getCountPlusOne'];
+ 获取 State：this.\$store.state.helloWorld.count;

#### 在 module 内获取总仓库的状态

```js
// store.js
// ...
export default {
  // ...
  getters: {
    /**
    * state、getters：当前模块的 state、getters
    * rootState、rootGetters：总仓库的状态
    */
    getTotalCount(state, getters, rootState, rootGetters) {
      return state.count + rootState.count;
    },
  },
  actions: {
    // 同上注释
    setTotalCount({ dispatch, commit, getters, state, rootGetters, rootState}) {
      const totalCount = state.count + rootState.count;
      commit('SET_COUNT', totalCount);
    },
  },
}
```

这样一来，我们一个健壮的`Vuex`共享数据仓库就建造完毕了，下面会介绍一些便捷的操作方法，也就是`Vuex`提供的辅助函数。

#### 在子 module 内派发总仓库的事件

有时候我们需要在子模块内派发一些全局的事件，那么可以通过分发 action 或者提交 mutation 的时候，将 { root: true } 作为第三个参数传递即可。

```js
// src/helloWorld/store.js
export default {
  // ...
  actions: {
    setCount({ commit, dispatch }) {
      commit('setCount', 1, { root: true });
      dispatch('setCountAsync', 2, { root: true });
    },
  },
};
```

### 4.5 辅助函数

当我们需要频繁使用 this.\$stoe.xxx 时，就老是需要写这么长一串，而且 this.\$store.commit('aaaaa/bbbbb/ccccc', params) 也非常的不优雅。`Vuex` 提供给了我们一些辅助函数来让我们写出更清晰明朗的代码。

#### mapState

```js
import { mapState } from 'vuex';

export default {
  // ...
  computed: {
    // 参数是数组
    ...mapState([
      'count', // 映射 this.count 为 this.$store.state.count
    ]),
    // 取模块内的数据
    ...mapState('helloWorld', {
      localCount: 'count', // 映射 this.localCount 为 this.$store.state.helloWorld.count
    }),
    // 少见写法
    ...mapState({
      count: (state, getters) => state.count,
    }),
  },
  created() {
    console.log(this.count); // 输出 this.$store.state.count 的数据
  },
};
```

#### mapGetters

使用f方式与 mapState 一模一样，毕竟只是对 state 做了一点点操作而已。

#### mapMutation

必须是同步函数，否则 mutation 无法正确触发回调函数。

```js
import { mapMutations } from 'vuex';

export default {
  // ...
  methods: {
    ...mapMutaions([
      'setCount', // 映射 this.setCount() 映射为 this.$store.commit('setCount');
    ]),
    // 带命名空间
    ...mapMutations('helloWorld', {
      setCountLocal: 'setCount', // 映射 this.setCountlocal() 为 this.$store.commit('helloWorld/setCount');
    }),
    // 少见写法
    ...mapMutaions({
      setCount: (commit, args) => commit('setCount', args),
    }),
  },
};
```

#### mapAction

与 mapMutation 的使用方式一模一样，毕竟只是对 mutation 做了一点点操作而已， 少见写法里 commit 换成了 dispatch。

### 4.6 其他补充

#### 动态模块

> 模块动态注册功能使得其他 Vue 插件可以通过在 store 中附加新模块的方式来使用 Vuex 管理状态

试想有一个通知组件，属于外挂组件，需要用到`Vuex`来管理数据，那么我们可以这样做：

+ 注册：this.$store.registerModule(moduleName)
+ 销毁：this.$store.unregisterModule(moduleName)

**store.js**:

```js
export const namespace = 'notice';
export const store = {
  namespaced: true,
  state: {
    count: 1,
  },
  mutations: {
    setCount(state, count) {
      state.count += count;
    },
  },
};
```

**Notice.vue:**

```vue
<template>
  <div>
    {{ count }}
    <button @click="handleBtnClick">add count</button>
  </div>
</template>

<script>
import { store, namespace } from './store';

export default {
  name: 'Notice',
  computed: {
    count() {
      return this.$store.state[namespace].count;
    },
  },
  beforeCreate() {
    // 注册 notice 模块
    this.$store.registerModule(namespace, store);
  },
  methods: {
    handleBtnClick() {
      this.$store.commit(`${namespace}/setCount`, 1);
    },
  },
//  beforeDestroy() {
//    // 销毁 notice 模块
//    this.$store.unregisterModule(namespace);
//  },
}  
</script>
```

#### 模块重用

有时候需要创建一个模块的多个实例，那 state 可能会造成混乱。我们可以类似 vue 中 data 的做法，将 state 写成函数返回对象的方式：

```js
export default {
//  old state
//  state: {
//    count: 0,
//  },
  
// new state
  state() {
    return {
      count: 0,
    };
  },
};
```

#### createNamespacedHelpers

在模块有命名空间的时候，我们在使用数据或者派发事件的时候需要在常量前加上命名空间的值，有些时候写起来也不是很舒服，`Vuex`提供了一个辅助方法`createNamespacedHelpers`，能帮助我们直接生成带命名空间的辅助函数。

```js
// old
import { mapState } from 'vuex';

export default {
  // ...
  computed: {
    ...mapState('helloWorld', [
      'count',
    ]);
  },
};


// use createNamespacedHelpers function
import { createNamespacedHelpers } from 'vuex';
const { mapState } = createNamespacedHelpers('helloWorld');

export default {
  // ...
  computed: {
    ...mapState([
      'count',
    ]);
  },
};
```

#### 插件 

在上面我们已经使用到了一个`vuex/dist/logger`插件，他可以帮助我们追踪到 mutaion 的每一次变化，并且在控制台打出，类似下图。

![image-20190916154825437](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-09-20-063356.jpg)

可以清晰地看到变化前和变化后的数据，进行对比。

插件还会暴露 mutaion 钩子，可以在插件内提交 mutaion 来修改数据。

更多神奇的操作可以参考[官网](https://vuex.vuejs.org/zh/guide/plugins.html)慢慢研究，这里不是重点不做更多介绍（其实是我想象不到要怎么用）。

#### 严格模式

````js
const store = new Vuex.Store({
  // ...
  strict: true
})
````

当开启严格模式，只要 state 变化不由 mutaion 触发，则会抛出错误，方便追踪。

**生产环境请关闭，避免性能损失。**

可以通过构建工具来帮助我们`process.env.NODE_ENV !== 'production'`。

#### 表单处理

当在表单中通过`v-model`使用`Vuex`数据时，会有一些意外情况发生，因为用户的修改并不是由 mutaion 触发，所以解决的问题是：使用带有`setter`的双向绑定计算属性。

```js
// template
<input v-model="message">
  
// script
export default {
  // ...
  computed: {
    message: {
      get () {
        return this.$store.state.obj.message
      },
      set (value) {
        this.$store.commit('updateMessage', value)
      },
    },
  },
};
```

## 总结

通过上面的一些例子，我们知道了如何来正确又优雅地管理我们的数据，如何快乐地编写`Vuex`。回到开头，如果你还没有理解那张图的话，不妨再把这个过程多看一下，然后再看看[Vuex 从使用到源码分析（下篇）]()更深入地了解`Vuex`。

