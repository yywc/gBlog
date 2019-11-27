## 前言

该系列分为上中下三篇：

+ [Vuex 从使用到原理分析（上篇）](https://github.com/yywc/gBlog/blob/master/%E5%89%8D%E7%AB%AF/%E6%A1%86%E6%9E%B6/Vuex%20%E4%BB%8E%E4%BD%BF%E7%94%A8%E5%88%B0%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%EF%BC%88%E4%B8%8A%E7%AF%87%EF%BC%89.md)：介绍 Vuex，以及 Vuex 的几种常见写法；
+ [Vuex 从使用到原理分析（中篇）](https://github.com/yywc/gBlog/blob/master/%E5%89%8D%E7%AB%AF/%E6%A1%86%E6%9E%B6/Vuex%20%E4%BB%8E%E4%BD%BF%E7%94%A8%E5%88%B0%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%EF%BC%88%E4%B8%AD%E7%AF%87%EF%BC%89.md)：分析 Vuex 的初始化以及模块获取安装；
+ [Vuex 从使用到原理分析（下篇）](https://github.com/yywc/gBlog/blob/master/%E5%89%8D%E7%AB%AF/%E6%A1%86%E6%9E%B6/Vuex%20%E4%BB%8E%E4%BD%BF%E7%94%A8%E5%88%B0%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%EF%BC%88%E4%B8%8B%E7%AF%87%EF%BC%89.md)：分析 Vuex 和 Store 中的一些方法包括辅助函数；

在前面我们还没有分析 store 是怎样提交 mutation，怎样分发 dispatch 的呢？这篇就是对这些方法以及 4 个辅助函数进行分析。

## 1. commit

commit 方法是唯一允许提交 mutation 来修改 state 的途径，它定义在`store.js`里：

```js
  commit (_type, _payload, _options) {
    const {
      type,
      payload,
      options
    } = unifyObjectStyle(_type, _payload, _options)

    const mutation = { type, payload }
    const entry = this._mutations[type]
    if (!entry) {
      if (process.env.NODE_ENV !== 'production') {
        console.error(`[vuex] unknown mutation type: ${type}`)
      }
      return
    }
    this._withCommit(() => {
      entry.forEach(function commitIterator (handler) {
        handler(payload)
      })
    })
    // ...不重要代码
  }
```

在前面我们记得`commit`与`dispatch`方法是有多种传参写法的，所以这里第一步就是整合我们所需要的参数，获取到 type、payload、与 options。

从`store._mutations`上获取 type 对应的回调函数，如果没有找到，则在开发环境抛出错误，最后通过`_withCommit`来执行 entry 里的函数来执行 mutations 里的函数修改 state。

## 2. dispatch

`dispatch`前半部分与`commit`相同，获取 type 与 payload 然后从`store._actions`上拿到我们定义的 actions 里的回调函数。

```js
  dispatch (_type, _payload) {
    // check object-style dispatch
    const {
      type,
      payload
    } = unifyObjectStyle(_type, _payload)

    const action = { type, payload }
    const entry = this._actions[type]
    if (!entry) {
      if (process.env.NODE_ENV !== 'production') {
        console.error(`[vuex] unknown action type: ${type}`)
      }
      return
    }

    // 下面省略了一些代码，相当于结果如下代码
    return entry.length > 1
      ? Promise.all(entry.map(handler => handler(payload)))
      : entry[0](payload)
  }
```

由于我们 actions 里的方法返回的是 Promise，所以在 map 后用 Promise.all 来异步执行，然后 return 结果。

## 3. mapState

在上篇的使用讲解中，我们知道`mapState`可以帮助我们在组件中更好的获取 state 里的数据，`mapState`定义在`helpers.js`中：

```js
export const mapState = normalizeNamespace();
```

一开始就是执行了`normalizeNamespace`方法，我们先来看一下：

```js
function normalizeNamespace (fn) {
  return (namespace, map) => {
    if (typeof namespace !== 'string') {
      map = namespace
      namespace = ''
    } else if (namespace.charAt(namespace.length - 1) !== '/') {
      namespace += '/'
    }
    return fn(namespace, map)
  }
}
```

这个函数就是对命名空间的一层封装，如果有字符串型的命名空间，则把命名空间格式化成 '/moduleA' 的形式，否则就将参数后移，置空第一个参数（命名空间），再返回回调函数执行的结果。

```js
export const mapState = normalizeNamespace((namespace, states) => {
  const res = {}
  normalizeMap(states).forEach()
  return res
})
```

噢，这里又来了一个`normalizeMap`方法，不得不看一下：

```js
function normalizeMap (map) {
  return Array.isArray(map)
    ? map.map(key => ({ key, val: key }))
    : Object.keys(map).map(key => ({ key, val: map[key] }))
}
```

很简单，就是格式化 map 为 [{ key, value }] 的形式，例如：

+ [1, 2] => [ { key: 1, value: 1 }, { key: 2, value: 2 } ]
+ { a: 1, b: 2 } => [{ key: 'a', value: 1 }, { key: 'b', value: 2 }]

为什么会有这一步，因为我们在 computed 里定义 mapState() 的时候，有下面两种写法：

```js
// 第一种
...mapState({
  countAlias: 'count',
  count: state => state.count,
}),
// 第二种
...mapState([
  'count',
]),
```

所以才有需要格式化参数的这一步。

接下来继续看：

```js
export const mapState = normalizeNamespace((namespace, states) => {
  const res = {}
  normalizeMap(states).forEach(({ key, val }) => {
    res[key] = function mappedState () {
      let state = this.$store.state
      let getters = this.$store.getters
      if (namespace) {
        const module = getModuleByNamespace(this.$store, 'mapState', namespace)
        if (!module) {
          return
        }
        state = module.context.state
        getters = module.context.getters
      }
      return typeof val === 'function'
        ? val.call(this, state, getters)
        : state[val]
    }
    // mark vuex getter for devtools
    res[key].vuex = true
  })
  return res
})
```

通过遍历格式化后的参数数组，给 res[key] 设置上 mappedState 函数，这个函数主要对 state、getters 设置正确的值（主要是是否有命名空间）。如果 val 是函数，则在此处执行，如果是非函数，则直接从 state 上取值。然后返回这个结果，最后返回 res 这个对象，所以我们在组件中使用的时候可以直接在`computed`里通过扩展运算符直接扩展到上面。

所以我们在组件中通过`this[key]`访问到的就是`this.$store.state[key]`或`this.$store.state[namespace][key]`的值了。

## 4. mapGetters

```js
export const mapGetters = normalizeNamespace((namespace, getters) => {
  const res = {}
  normalizeMap(getters).forEach(({ key, val }) => {
    // The namespace has been mutated by normalizeNamespace
    val = namespace + val
    res[key] = function mappedGetter () {
      if (namespace && !getModuleByNamespace(this.$store, 'mapGetters', namespace)) {
        return
      }
      if (process.env.NODE_ENV !== 'production' && !(val in this.$store.getters)) {
        console.error(`[vuex] unknown getter: ${val}`)
        return
      }
      return this.$store.getters[val]
    }
    // mark vuex getter for devtools
    res[key].vuex = true
  })
  return res
})
```

与`mapState`类似，也是先定义 res 空对象，然后在 res[key] 上挂载方法，在方法内部判断命名空间，返回 store 上 getters 对应的值，最后返回这个 res 对象。

## 5. mapMutations

```js
export const mapMutations = normalizeNamespace((namespace, mutations) => {
  const res = {}
  normalizeMap(mutations).forEach(({ key, val }) => {
    res[key] = function mappedMutation (...args) {
      // Get the commit method from store
      let commit = this.$store.commit
      if (namespace) {
        const module = getModuleByNamespace(this.$store, 'mapMutations', namespace)
        if (!module) {
          return
        }
        commit = module.context.commit
      }
      return typeof val === 'function'
        ? val.apply(this, [commit].concat(args))
        : commit.apply(this.$store, [val].concat(args))
    }
  })
  return res
})
```

前面都一样，都是格式化参数，然后从 store/module 上拿到 commit 方法，再判断 val 是不是函数，最终通过 store.commit 方法来修改 state 的值。

## 6. mapActions

```js
export const mapActions = normalizeNamespace((namespace, actions) => {
  const res = {}
  normalizeMap(actions).forEach(({ key, val }) => {
    res[key] = function mappedAction (...args) {
      // get dispatch function from store
      let dispatch = this.$store.dispatch
      if (namespace) {
        const module = getModuleByNamespace(this.$store, 'mapActions', namespace)
        if (!module) {
          return
        }
        dispatch = module.context.dispatch
      }
      return typeof val === 'function'
        ? val.apply(this, [dispatch].concat(args))
        : dispatch.apply(this.$store, [val].concat(args))
    }
  })
  return res
})
```

与`mapMutations`几乎一样，`commit`换成了`dispatch`而已。

## 7. registerModule

```js
registerModule (path, rawModule, options = {}) {
  if (typeof path === 'string') path = [path]

  if (process.env.NODE_ENV !== 'production') {
    assert(Array.isArray(path), `module path must be a string or an Array.`)
    assert(path.length > 0, 'cannot register the root module by using registerModule.')
  }

  this._modules.register(path, rawModule)
  installModule(this, this.state, path, this._modules.get(path), options.preserveState)
  // reset store to update getters...
  resetStoreVM(this, this.state)
}
```

看过中篇里的分析，这个方法就很简单了。首先是格式化 path 参数，path 可以传入字符串和数组，传数组时是注册一个带命名空间的模块，统一格式化成数组形式。

调用 register 方法 => 安装模块 => 初始化 storm.\_.vm。

## 8. unregisterModule

```js
unregisterModule (path) {
  if (typeof path === 'string') path = [path]

  if (process.env.NODE_ENV !== 'production') {
    assert(Array.isArray(path), `module path must be a string or an Array.`)
  }

  this._modules.unregister(path)
  this._withCommit(() => {
    const parentState = getNestedState(this.state, path.slice(0, -1))
    Vue.delete(parentState, path[path.length - 1])
  })
  resetStore(this)
}
```

首先也是格式化 path 参数为数组，然后调用`unregister`方法，这个方法定义在`module/module-collectionjs`里。

```js
unregister (path) {
  const parent = this.get(path.slice(0, -1))
  const key = path[path.length - 1]
  if (!parent.getChild(key).runtime) return

  parent.removeChild(key)
}
```

这个方法就是层层取值，找到该模块的父模块，然后从父模块的 \_childrent 对象中移除对应的属性。

调用`unregister`后再通过`_withcommit`方法，将 state 从父模块的 state 中移除。

最后最后调用`resetStore`方法重置 store。

```js
function resetStore (store, hot) {
  store._actions = Object.create(null)
  store._mutations = Object.create(null)
  store._wrappedGetters = Object.create(null)
  store._modulesNamespaceMap = Object.create(null)
  const state = store.state
  // init all modules
  installModule(store, state, [], store._modules.root, true)
  // reset vm
  resetStoreVM(store, state, hot)
}
```

这里就主要是重置几个核心概念，然后重新安装模块，重新初始化 store.\_vm。

## 9. createNamespacedHelpers

```js
export const createNamespacedHelpers = (namespace) => ({
  mapState: mapState.bind(null, namespace),
  mapGetters: mapGetters.bind(null, namespace),
  mapMutations: mapMutations.bind(null, namespace),
  mapActions: mapActions.bind(null, namespace)
})
```

这个函数太简单了，就是返回一个对象，这个对象包含 4 个传入了命名空间的辅助函数。

## 总结

到这里为止，整个`Vuex`的核心概念以及运行原理我们都已经分析完了。理解了这些，我们也能更好地在平时开发中定位错误，像里面一些`normalizeNamespace`、`normalizeMap`等方法也是我们能在平时开发中学习使用的。
