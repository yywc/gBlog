## 前言

该系列分为上中下三篇：

+ [Vuex 从使用到原理分析（上篇）](https://github.com/yywc/gBlog/blob/master/%E5%89%8D%E7%AB%AF/%E6%A1%86%E6%9E%B6/Vuex%20%E4%BB%8E%E4%BD%BF%E7%94%A8%E5%88%B0%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%EF%BC%88%E4%B8%8A%E7%AF%87%EF%BC%89.md)：介绍 Vuex，以及 Vuex 的几种常见写法；
+ [Vuex 从使用到原理分析（中篇）](https://github.com/yywc/gBlog/blob/master/%E5%89%8D%E7%AB%AF/%E6%A1%86%E6%9E%B6/Vuex%20%E4%BB%8E%E4%BD%BF%E7%94%A8%E5%88%B0%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%EF%BC%88%E4%B8%AD%E7%AF%87%EF%BC%89.md)：分析 Vuex 的初始化以及模块获取安装；
+ [Vuex 从使用到原理分析（下篇）](https://github.com/yywc/gBlog/blob/master/%E5%89%8D%E7%AB%AF/%E6%A1%86%E6%9E%B6/Vuex%20%E4%BB%8E%E4%BD%BF%E7%94%A8%E5%88%B0%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%EF%BC%88%E4%B8%8B%E7%AF%87%EF%BC%89.md)：分析 Vuex 和 Store 中的一些方法包括辅助函数；

在上一篇中，我们大致了解了`Vuex`的概念以及食用方式，这里主要是从源码的角度来详细揭开`Vuex`的神秘面纱。

我们以入门版食用方式加一些模块为例：

```js
// store.js
import Vue from 'vue';
import Vuex from 'vuex';

Vue.use(Vuex);
const vm = new Vue();

// module a
const moduleA = {
  namespaced: true,
  state: {
    number: 0,
  },
  getters: {
    getNumberPlusOne(state) {
      return state.number + 1;
    },
  },
  mutations: {
    setNumber(state, num) {
      state.number = num;
    },
  },
  actions: {
    async setNumberAsync({ commit }) {
      const { data } = await vm.$http('/api/get-number');
      commit('setNumber', data);
    },
  },
};

// main
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
    async setCountAsync({ commit }, count) {
      const { data: count } = await vm.$http('/api/example', { count });
      commit('setCount', count);
    },
  },
  modules: {
    moduleA,
  },
});
```

下面一步步分析。

**PS：未特殊说明路径，代码内容则在当前文件内。**

## 1. import Vuex from 'vuex'

打开 vuex 源码，我们这里采用的是 3.1.1 版本进行分析，找到 `index.js`，内容如下：

```js
import { Store, install } from './store'
import { mapState, mapMutations, mapGetters, mapActions, createNamespacedHelpers } from './helpers'

export default {
  Store,
  install,
  version: '__VERSION__',
  mapState,
  mapMutations,
  mapGetters,
  mapActions,
  createNamespacedHelpers
}
```

由此可见，我们引入 Vuex 对象，包含了辅助函数以及 Store 对象、install 方法以及 createNamespacedHelpers 辅助函数。

## 2. Vue.use(Vuex)

我们知道 Vue.use 方法会注入一个插件，调用这个对象的`install`方法，我们打开 `store.js`找到最后的`install`方法：

```js
export function install (_Vue) {
  if (Vue && _Vue === Vue) {
    if (process.env.NODE_ENV !== 'production') {
      console.error(
        '[vuex] already installed. Vue.use(Vuex) should be called only once.'
      )
    }
    return
  }
  Vue = _Vue
  applyMixin(Vue)
}
```

1. 对 Vue 对象做了一层校验，防止重复安装
2. 将外层 Vue 赋值给本地 Vue
3. 调用 applyMixin 方法初始化 vuex store

`mixin.js 中的 applyMixin 方法：`

我们精简一下，核心内容如下：

```js
export default function (Vue) {

  Vue.mixin({ beforeCreate: vuexInit })

  function vuexInit () {
    const options = this.$options
    // store injection
    this.$store = typeof options.store === 'function'
      ? options.store()
      : options.store
  }
}
```

这样看来，其实就是在 Vue beforeCreate 生命周期钩子函数里执行了 vuexInit 方法，将实例化的 Store 对象挂载到 \$store 上，这也是为什么我们能在 vue 组件中直接通过 this.\$store 就可以进行相关操作的原因。

## 3. new Vuex.Store()

当我们实例化一个 Store 的时候，进行了什么操作呢？

我们找到`store.js`里的 Store 类，省略一些不太重要的代码，主要看到如下部分：

我们省略一些不太重要的代码，主要看到如下部分：

```js
export class Store {
  constructor (options = {}) {
    if (!Vue && typeof window !== 'undefined' && window.Vue) {
      install(window.Vue)
    }
    if (process.env.NODE_ENV !== 'production') {
      assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
      assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
      assert(this instanceof Store, `store must be called with the new operator.`)
    }

    // ...

    this._modules = new ModuleCollection(options)

    // ...
    const state = this._modules.root.state
    installModule(this, state, [], this._modules.root)

    resetStoreVM(this, state)
  }
}
```

上面几个部分也就是主要的逻辑

1. 安装 Vuex 插件
2. 获取模块
3. 初始化 store vm

首先是如果没有传入 Vue，那么自动安装一下插件，如果是开发环境则会报错。

store 可以拆分成各种小的 store，那么整个看起来就是一个树形结构。store 本身相当于根节点，每一个 module 都是一个子节点，那么首先要做的就是获取到这些子节点。

之后再将子节点里的数据以及 getter、state 进行关联。

## 4. 获取模块

我们打开`module/module-collection.js`文件，找到 ModuleCollection 类，如下：

```js
export default class ModuleCollection {
  constructor (rawRootModule) {
    this.register([], rawRootModule, false)
  }
}
```

可以看到实例化的过程就是执行了 register 方法。

```js
register (path, rawModule, runtime = true) {
  const newModule = new Module(rawModule, runtime)
  if (path.length === 0) {
    this.root = newModule
  } else {
    const parent = this.get(path.slice(0, -1))
    parent.addChild(path[path.length - 1], newModule)
  }

  if (rawModule.modules) {
    forEachValue(rawModule.modules, (rawChildModule, key) => {
      this.register(path.concat(key), rawChildModule, runtime)
    })
  }
}
```

+ path：module 的路径
+ rowModule：export default 的对象，模块的配置项
+ runtime：是否是运行时创建的模块

这里第一步是实例化了一个 Module 对象，这个类定义在 `module/module.js`里：

```js
export default class Module {
  constructor (rawModule, runtime) {
    this.runtime = runtime

    this._children = Object.create(null)

    this._rawModule = rawModule
    const rawState = rawModule.state

    this.state = (typeof rawState === 'function' ? rawState() : rawState) || {}
  }
}
```

\_children 是该模块的子模块，\_rawModule 是该模块的配置，state 是该模块的 state。

返回到 register 方法继续，实例化 Module 后，接下来第一次进入 path 是空数组，所以这就是根 store，赋值当前配置到 root 上。

接着判断是否有 modules 项，如果有，则执行下面代码：

```js
forEachValue(rawModule.modules, (rawChildModule, key) => {
  this.register(path.concat(key), rawChildModule, runtime)
})
```

这段代码主要是遍历 modules，递归调用 register 方法，将配置项里的 modules 的 key 作为路径保存到 path 中，传入子 module 和创建状态。（以开始的例子，key 就是 'moduleA'）

第二次进入 register，此时走到`if (path.length === 0) {}`的判断，由于此时 path 已经有内容了，所以会执行 else 的逻辑：

```js
const parent = this.get(path.slice(0, -1)) // 这里相当于 path 弹出了最后一项
parent.addChild(path[path.length - 1], newModule)


get (path) {
  return path.reduce((module, key) => {
    return module.getChild(key)
  }, this.root)
}

// module.js 里的 addChild、getChild
addChild (key, module) {
  this._children[key] = module
}
getChild (key) {
  return this._children[key]
}
```

首先获取父模块，这里通过 get 方法中的 reduce，层层递进深度搜索出当前模块的父模块然后返回。

通过 Module 实例的 addChild 方法给挂载到 _children 上，（例子中相当于 key: 'moduleA', value: moduleA 对象）。

这样递归注册，就对所有的模块进行实例化，通过 _children 建立好父子关系，一颗组件树就构建完成了。

## 5. 安装模块

当我们构建好模块树，接下来就需要去安装这些模块了，截取 installModule 方法代码如下：

```js
function installModule (store, rootState, path, module, hot) {
  const isRoot = !path.length
  const namespace = store._modules.getNamespace(path)

  // register in namespace map
  if (module.namespaced) {
    store._modulesNamespaceMap[namespace] = module
  }

  // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      Vue.set(parentState, moduleName, module.state)
    })
  }

  const local = module.context = makeLocalContext(store, namespace, path)

  module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key
    registerMutation(store, namespacedType, mutation, local)
  })

  module.forEachAction((action, key) => {
    const type = action.root ? key : namespace + key
    const handler = action.handler || action
    registerAction(store, type, handler, local)
  })

  module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key
    registerGetter(store, namespacedType, getter, local)
  })

  module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot)
  })
}
```

这里的主要逻辑就是初始化 state、getters、mutations、actions，这里有 5 个参数，分别代表以下意思：

+ store：root store
+ rootState：root state
+ path：模块访问路径
+ module：当前模块
+ hot：是否热更新

**第一步**定义 isRoot 变量用来判断是否是 root store，接下来获取我们定义的命名空间，如果有定义命名空间（namespaced: true），则把模块挂载到以命名空间为 key 的 \_modulesNamespaceMap 对象上。

**第二步**判断非根模块非热更新的情况下，获取父模块的 state，获取当前模块的名称，通过 Vue.set 将当前模块的 state 挂载到父模块上，key 是模块名称。所以这也是 store 里的数据都是响应式的原因。

```js
store._withCommit(() => {
  Vue.set(parentState, moduleName, module.state)
})


_withCommit (fn) {
  const committing = this._committing
  this._committing = true
  fn()
  this._committing = committing
}

// 后面会介绍（在安装模式的严格模式中）
if (process.env.NODE_ENV !== 'production') {
  assert(store._committing, `Do not mutate vuex store state outside mutation handlers.`)
}
```

通过 \_withCommit 的代理，我们在修改 state 的时候，在开发环境通过 this._committing 标志就能抛出错误，避免意外更改。

**第三步**通过`makeLocalContext`方法创建本地上下文环境，接收 store（root store）、namespace（模块命名空间）、path（模块路径） 三个参数。

```js
function makeLocalContext (store, namespace, path) {
  const noNamespace = namespace === ''

  const local = {
    dispatch: noNamespace ? store.dispatch : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args

      if (!options || !options.root) {
        type = namespace + type
        if (process.env.NODE_ENV !== 'production' && !store._actions[type]) {
          console.error(`[vuex] unknown local action type: ${args.type}, global type: ${type}`)
          return
        }
      }

      return store.dispatch(type, payload)
    },

    commit: noNamespace ? store.commit : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args

      if (!options || !options.root) {
        type = namespace + type
        if (process.env.NODE_ENV !== 'production' && !store._mutations[type]) {
          console.error(`[vuex] unknown local mutation type: ${args.type}, global type: ${type}`)
          return
        }
      }

      store.commit(type, payload, options)
    }
  }

  Object.defineProperties(local, {
    getters: {
      get: noNamespace
        ? () => store.getters
        : () => makeLocalGetters(store, namespace)
    },
    state: {
      get: () => getNestedState(store.state, path)
    }
  })

  return local
}
```

这里没有命名空间的情况就是直接使用 store 上的 dispatch 和 commit，如果有则使用新定义的方法，这个方法接收三个参数：

+ _type：dispatch、commit 的 type
+ _payload：提交的参数
+ _options：其他选项，例如 { root: true } 这个在子模块派发到根仓库的配置项

我们在 commit、dispatch 的时候会有两种写法传参：

1. type, payload, options
2. { type, payload }, options

`unifyObjectStyle 函数`

```js
function unifyObjectStyle (type, payload, options) {
  if (isObject(type) && type.type) { // 如果是对象传参
    options = payload // 第二个参数是 { root: true } 这种了
    payload = type // 第一个参数就是 payload，不管里面的 type 属性
    type = type.type // 将 type 值赋值给 type
  }

  // 对 type 类型的一个断言
  if (process.env.NODE_ENV !== 'production') {
    assert(typeof type === 'string', `expects string as the type, but found ${typeof type}.`)
  }

  // 返回一个完整对象
  return { type, payload, options }
}
```

所以 unifyObjectStyle 函数就是帮助我们把参数整合成 type、payload、options 三个变量里。

接下来判断只要不是派发到跟模块或者当前模块就是根模块，那么 type 就需要加上命名空间（例子中就变成了 'moduleA/setNumber'），然后 commit/dispatch 出去。

最后将 getters、state 通过 defineProperties 劫持到 local 对象上，值为当前模块的 getters、state。

`makeLocalGetters、getNestedState 函数`

```js
function makeLocalGetters (store, namespace) {
  const gettersProxy = {}

  const splitPos = namespace.length
  Object.keys(store.getters).forEach(type => {

    // 判断 type 前的命名空间是否匹配当前模块的命名
    // 例子中 type 是 'moduleA/getNumberPlusOne', namespace 是 'moduleA/'
    if (type.slice(0, splitPos) !== namespace) return

    // 获取本地 type，也就是 getNumberPlusOne
    const localType = type.slice(splitPos)

    // 这一步使得 localType 实际上就是访问了 store.getters[type]
    Object.defineProperty(gettersProxy, localType, {
      get: () => store.getters[type],
      enumerable: true
    })
  })

  // 访问代理对象
  return gettersProxy
}

// 通过 reduce 一层层获取到当前模块的 state，然后返回这个 state
function getNestedState (state, path) {
  return path.length
    ? path.reduce((state, key) => state[key], state)
    : state
}
```

获取进行了各种代理数据的本地上下文后，接下来会遍历 mutations、actions、getters，分别进行注册，而 state 的注册早就在之前实例化 Module 的时候就完成了。

### Mutations 注册

```js
module.forEachMutation((mutation, key) => {
  const namespacedType = namespace + key
  registerMutation(store, namespacedType, mutation, local)
})
```

mutations 的注册相对简单，遍历 module 下的每一个 mutations 属性的值，然后获取带有命名空间的 type，再调用 registerMutation 方法进行注册，传入4个参数：

+ store：store 实例
+ namespacedType：带命名空间的 type。例子中是 'moduleA/setNumber'
+ handler：type 处理函数，也就是 setNumber 函数
+ local：上下文环境，root 为 store，module 为 local

`registerMutation 函数：`

```js
function registerMutation (store, type, handler, local) {
  const entry = store._mutations[type] || (store._mutations[type] = [])
  entry.push(function wrappedMutationHandler (payload) {
    handler.call(store, local.state, payload)
  })
}
```

entry 是一个数组，为什么是数组呢，当我们没有使用命名空间时，恰巧在子模块也有一个`setCount`方法，那么这个方法就会存到 setCount 为属性值的一个数组中，从而允许我们一个 type 对应多个 mutaions。

entry push 一个执行 type 的回调函数的一个包装函数，这也是 mutaions 里的函数支持两个参数 state、payload 的原因。

### Actions 注册

```js
  module.forEachAction((action, key) => {
    const type = action.root ? key : namespace + key
    const handler = action.handler || action
    registerAction(store, type, handler, local)
  })
```

回调主要做了三件事：

1. 获取 action type
2. 获取 action 回调
3. 调用 registerAction 注册方法

`registerAction 方法：`

```js
function registerAction (store, type, handler, local) {
  const entry = store._actions[type] || (store._actions[type] = [])
  entry.push(function wrappedActionHandler (payload, cb) {
    let res = handler.call(store, {
      dispatch: local.dispatch,
      commit: local.commit,
      getters: local.getters,
      state: local.state,
      rootGetters: store.getters,
      rootState: store.state
    }, payload, cb)
    if (!isPromise(res)) {
      res = Promise.resolve(res)
    }
    if (store._devtoolHook) {
      return res.catch(err => {
        store._devtoolHook.emit('vuex:error', err)
        throw err
      })
    } else {
      return res
    }
  })
}
```

忽略几个 if 的判断，可以看到 actions 里通过 Promise 实现异步过程，这也是为什么 mutaions 里不支持异步，而可以通过 actions 来完成了。

actions 回调的第一个参数是一个对象，里面包含 dispatch、commit、getters、state、rootGetters、rootState 字段，第二个参数一般是我们所传递的参数，**第三个参数 cb 经本人验证完全没有用，在`dispatch`方法中 handler 也只提供了 payload 一个参数**，最后根据 res 的类型返回对应的值。

### Getters 注册

```js
  module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key
    registerGetter(store, namespacedType, getter, local)
  })
```

这里没什么好说的，看看`registerGetter`方法：

```js
function registerGetter (store, type, rawGetter, local) {
  if (store._wrappedGetters[type]) {
    if (process.env.NODE_ENV !== 'production') {
      console.error(`[vuex] duplicate getter key: ${type}`)
    }
    return
  }
  store._wrappedGetters[type] = function wrappedGetter (store) {
    return rawGetter(
      local.state, // local state
      local.getters, // local getters
      store.state, // root state
      store.getters // root getters
    )
  }
}
```

首先抛个错，防止 getter key 重复，接着在 \_wrappedGetters 上以 type 为 key，挂载 wrappedGetter 函数，返回 rawGetters 函数执行的结果，这个函数就是我们定义在 store.js getters 里 type 对应的回调。

所以我们在 getters 里定义函数接收的参数有 state、getters、rootState、rootGetters 4个，在[Vuex 从使用到原理分析（上篇）](https://github.com/yywc/gBlog/blob/master/%E5%89%8D%E7%AB%AF/%E6%A1%86%E6%9E%B6/Vuex%20%E4%BB%8E%E4%BD%BF%E7%94%A8%E5%88%B0%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%EF%BC%88%E4%B8%8A%E7%AF%87%EF%BC%89.md)的高级版食用方式中有说明应用。

### 安装子模块

当上面步骤都完成后，就开始遍历模块的子模块，然后递归安装。

```js
  module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot)
  })
```

## 6. 初始化 store._vm

```js
function resetStoreVM (store, state, hot) {
  const oldVm = store._vm

  store.getters = {}
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  forEachValue(wrappedGetters, (fn, key) => {
    computed[key] = partial(fn, store)
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
  })

  const silent = Vue.config.silent
  Vue.config.silent = true
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
  Vue.config.silent = silent

  if (store.strict) {
    enableStrictMode(store)
  }

  if (oldVm) {
    if (hot) {
      store._withCommit(() => {
        oldVm._data.$$state = null
      })
    }
    Vue.nextTick(() => oldVm.$destroy())
  }
}
```

这里主要是将 state 与 getters 建立好关系，实例化一个 Vue 挂载到 \_vm 属性上，通过 computed 属性将 getters 与 state 关联起来并缓存结果。我们访问`this.$store.getters.getCountPlusOne`的时候，其实访问的就是`this.$store._vm.getCountPlusOne`，再继续就是访问到的 \_vm 的 computed 里定义的数据，

在执行`computed.getCountPlusOne`对应的函数时，会执行`store._wrappedGetters.getCountPlusOne`方法，这个方法又是我们在分析注册 Getters 时有提到的`wrappedGetter`的方法：

```js
function registerGetter (store, type, rawGetter, local) {
  // ...
  store._wrappedGetters[type] = function wrappedGetter (store) {
    return rawGetter(
      local.state, // local state
      local.getters, // local getters
      store.state, // root state
      store.getters // root getters
    )
  }
}
```

所以最后是执行了我们定义的 getters 对象里的方法，这里就会访问到`store.state`，进而访问到`store._vm._data.$$state`，通过这样一层一层，就建立了 state 与 getters 的依赖关系，当`store.state`的发生变化时，下次访问`store.getters`就获得重新计算的结果，我们用一张图来更为直观的看清楚这个过程。

![getetr 与 state](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100601.jpg)

接下来的就相对简单了，一个是严格模式，一个是销毁旧的实例。下面看看严格模式：

```js
if (store.strict) {
  enableStrictMode(store)
}

function enableStrictMode (store) {
  store._vm.$watch(function () { return this._data.$$state }, () => {
    if (process.env.NODE_ENV !== 'production') {
      assert(store._committing, `Do not mutate vuex store state outside mutation handlers.`)
    }
  }, { deep: true, sync: true })
}
```

在安装模块中有提到 [_commit](#5. 安装模块) 方法，里面有个`_committing`字段，就是在这里使用到的。

严格模式中，`_vm`会`watch` `$$state`的变化，当`store.state`变化时，`_committing`必须为`true`，否则在开发环境抛出警告。而`_committing`的值只会在`_commit`方法中提`mutaion`时会被短暂置为`true`，所以`Vuex`通过这种操作来规避我们在其他地方修改了`store.state`，而没有按照预期。

## 总结

这一篇主要介绍了引入 Vuex，注册 Store，获取模块，构建模块树，安装模块以及给模块注册 mutaions、actions、getters，最后通过 store._vm 给 state 与 getters 绑定，以及通过 computed 来缓存 getters 的结果。但是还有一个方法我们没有说明，例如 commit、dispatch 以及 4 个辅助函数，这些内容会在[Vuex 从使用到原理分析（下篇）](https://github.com/yywc/gBlog/blob/master/%E5%89%8D%E7%AB%AF/%E6%A1%86%E6%9E%B6/Vuex%20%E4%BB%8E%E4%BD%BF%E7%94%A8%E5%88%B0%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%EF%BC%88%E4%B8%8B%E7%AF%87%EF%BC%89.md)中进行分析。
