## 前言

在上篇中我们分析到 Vue.use 时，里面调用了 init 方法，该方法中有一段代码控制了路由的跳转：

```js
history.transitionTo(
  history.getCurrentLocation(),
  setupHashListener,
  setupHashListener
)
```

就是这个`transitionTo`方法决定了我们路由跳转时的逻辑，回顾一下定义的路由：

```js
export default new Router({
  routes: [
    {
      path: '/foo',
      name: 'Foo',
      component: Foo,
      meta: { permission: true },
      children: [
        {
          path: 'child/:id',
          name: 'Child',
          component: Child,
        },
      ],
    },
    {
      path: '/bar',
      name: 'Bar',
      component: Bar,
      meta: { permission: true },
      beforeEnter: (to, from, next) => {
        console.log('Bar router beforeEnter');
        next();
      },
    },
  ],
});
```

现在以`http://localhost:8080/#/foo`跳转到`http://localhost:8080/#/foo/child/10000`为例。

## 1. transitionTo 函数

当我们点击页面上的 to-child 时，会触发点击事件从而调用`HashHistory`的`push`方法来调用`transitionTo`方法。

`transtionTo`函数定义在`src/history/base.js`里：

```js
transitionTo (
  // location = { _normalize: true, path: '/foo/child/10000', params: { id: 10000 } }
  location: RawLocation, 
  // onComplete = route => {
  //   pushHash(route.fullPath)
  //   handleScroll(this.router, route, fromRoute, false)
  //   onComplete && onComplete(route)
  // }
  onComplete?: Function,
  onAbort?: Function // onAbort = undefined
) {
  // this.current = {
  //   fullPath: "/foo",
  //   hash: "",
  //   matched: [{…}],
  //   meta: {permission: true},
  //   name: "Foo",
  //   params: {},
  //   path: "/foo",
  // }
  // 会调用 this.matcher.match 也就是 createMatcher 中的 match 方法找到匹配的 Route 对象
  // route = {
  //   fullPath: '/foo/child/10000',
  //   hash: '',
  //   matched: (2) [{…}, {…}],
  //   meta: {},
  //   name: 'Child',
  //   params: { id: '10000' },
  //   path: '/foo/child/10000',
  //   query: {},
  // }
  const route = this.router.match(location, this.current)
  this.confirmTransition(
    route,
    () => {
      this.updateRoute(route)
      onComplete && onComplete(route)
      this.ensureURL()

      // fire ready cbs once
      if (!this.ready) {
        this.ready = true
        this.readyCbs.forEach(cb => {
          cb(route)
        })
      }
    },
    err => {
      if (onAbort) {
        onAbort(err)
      }
      if (err && !this.ready) {
        this.ready = true
        this.readyErrorCbs.forEach(cb => {
          cb(err)
        })
      }
    }
  )
}
```

## 2. confirmTransition 函数

从上面的代码可以看到，当通过 matcher 匹配到 route 后，会执行`confirmTransition`方法，这个方法就定义在下面：

```js
confirmTransition (route: Route, onComplete: Function, onAbort?: Function) {
  const current = this.current
  const abort = err => {
    // after merging https://github.com/vuejs/vue-router/pull/2771 we
    // When the user navigates through history through back/forward buttons
    // we do not want to throw the error. We only throw it if directly calling
    // push/replace. That's why it's not included in isError
    if (!isExtendedError(NavigationDuplicated, err) && isError(err)) {
      if (this.errorCbs.length) {
        this.errorCbs.forEach(cb => {
          cb(err)
        })
      } else {
        warn(false, 'uncaught error during route navigation:')
        console.error(err)
      }
    }
    onAbort && onAbort(err)
  }
  if (
    isSameRoute(route, current) &&
    // in the case the route map has been dynamically appended to
    route.matched.length === current.matched.length
  ) {
    this.ensureURL()
    return abort(new NavigationDuplicated(route))
  }
  // ...
}
```

首先将 this.current 的值赋值到 current  上，这里也就` { path: '/foo', name: 'Foo', ... }`对象了。接着定义 abort 函数，判断一些情况输出错误信息。

接下来判断路径是否是同一条，调用 ensureURL 方法以及返回 abort 函数执行结果。

```js
ensureURL (push?: boolean) {
  const current = this.current.fullPath
  if (getHash() !== current) {
    push ? pushHash(current) : replaceHash(current)
  }
}
```

### 2.1 resolveQueue

执行`resolveQueue`方法得到`updated, deactivated, activated`三个值。

```js
function resolveQueue (
  current: Array<RouteRecord>, // current = [{ path: '/foo', ... }]
  next: Array<RouteRecord> // next = [{ path: '/foo', ... }, { path: '/foo/child/:id' }]
): {
  updated: Array<RouteRecord>,
  activated: Array<RouteRecord>,
  deactivated: Array<RouteRecord>
} {
  let i // i = undefined
  const max = Math.max(current.length, next.length) // max = 2
  // 经过循环，找到 cuurent 不等于 next 的那个索引 i，也就是匹配到的 child，i = 1
  for (i = 0; i < max; i++) {
    if (current[i] !== next[i]) {
      break
    }
  }
  // 以 i 为索引，next 分割出已更新的和激活的，而通过 current 分割出失活的。此处是 []
  // 当 /foo/child/:id 到 /foo 时，那么 deactivated 则是 [{ path: '/foo/child/:id', ... }] 了
  return {
    updated: next.slice(0, i),
    activated: next.slice(i),
    deactivated: current.slice(i)
  }
}
```

这个方法主要是获得获得更新的 record，激活的 record 以及失活的 record。

### 2.2 runQueue

下面`runQueue`部分的代码稍稍有点复杂，单独拎出来介绍一下。

```js
confirmTransition (route: Route, onComplete: Function, onAbort?: Function) {
  // ... 接上面省略号部分
  
  const queue: Array<?NavigationGuard> = [].concat(
    // in-component leave guards
    extractLeaveGuards(deactivated),
    // global before hooks
    this.router.beforeHooks,
    // in-component update hooks
    extractUpdateHooks(updated),
    // in-config enter guards
    activated.map(m => m.beforeEnter),
    // async components
    resolveAsyncComponents(activated)
  )

  this.pending = route
  const iterator = (hook: NavigationGuard, next) => {
    if (this.pending !== route) {
      return abort()
    }
    try {
      hook(route, current, (to: any) => {
        if (to === false || isError(to)) {
          // next(false) -> abort navigation, ensure current URL
          this.ensureURL(true)
          abort(to)
        } else if (
          typeof to === 'string' ||
          (typeof to === 'object' &&
            (typeof to.path === 'string' || typeof to.name === 'string'))
        ) {
          // next('/') or next({ path: '/' }) -> redirect
          abort()
          if (typeof to === 'object' && to.replace) {
            this.replace(to)
          } else {
            this.push(to)
          }
        } else {
          // confirm transition and pass on the value
          next(to)
        }
      })
    } catch (e) {
      abort(e)
    }
  }

  runQueue(queue, iterator, () => {
    const postEnterCbs = []
    const isValid = () => this.current === route
    // wait until async components are resolved before
    // extracting in-component enter guards
    const enterGuards = extractEnterGuards(activated, postEnterCbs, isValid)
    const queue = enterGuards.concat(this.router.resolveHooks)
    runQueue(queue, iterator, () => {
      if (this.pending !== route) {
        return abort()
      }
      this.pending = null
      onComplete(route)
      if (this.router.app) {
        this.router.app.$nextTick(() => {
          postEnterCbs.forEach(cb => {
            cb()
          })
        })
      }
    })
  }) 
}
```

首先定义了一个`queue`导航守卫相关的数组，接着定义了一个迭代器`iterator`，接收一个`NavigationGuard`型的函数以及`next`回调函数，我们简单看一下`NavigationGuard`函数类型：

```ts
declare type NavigationGuard = (
  to: Route,
  from: Route,
  next: (to?: RawLocation | false | Function | void) => void
) => any
```

可以发现，这个函数和我们在 demo 中的`main.js`里定义的 router 回调函数一毛一样！是不是很熟悉，是不是像看到了久别的亲人！好了，继续往下看。

然后执行`runQueue`这个方法，传入`queue`数组、`iterator`迭代器以及一个回调函数，查看一下`runQueue`函数了解一下，这里也是一个可以学习的编程技巧，它定义在`src/util/async.js`里面：

```js
/* @flow */

export function runQueue (queue: Array<?NavigationGuard>, fn: Function, cb: Function) {
  const step = index => {
    if (index >= queue.length) {
      cb()
    } else {
      if (queue[index]) {
        fn(queue[index], () => {
          step(index + 1)
        })
      } else {
        step(index + 1)
      }
    }
  }
  step(0)
}
```

接收是上面三个参数，然后定义了一个`step`函数，然后执行`step`，传入索引 0。

`step`函数第一步就判断索引是否超出`queue`边界，是则直接执行回调函数，否则判断`queue`对应索引是否存在，不存在则继续往下执行`step`。存在则执行`fn`，也就是我们的迭代器，并且回调函数是执行下一个索引的`step`，这也是`iterator`里的`next`函数。

**放到应用代码层面就是我们定义守卫钩子的时候，为什么一定要执行`next`的原因，不然`step`不执行路由也就没法继续了。**

为什么要这样做呢，因为`queue`里可能有异步组件，这样保证了按序执行。

### 2.3 queue 中的导航守卫

知道了`runQueue`的逻辑后，我们仔细看看到底 run 了哪些 queue 呢？

```js
const queue: Array<?NavigationGuard> = [].concat( // concat 会拍平里面的数组
  // in-component leave guards
  extractLeaveGuards(deactivated),
  // global before hooks
  this.router.beforeHooks,
  // in-component update hooks
  extractUpdateHooks(updated),
  // in-config enter guards
  activated.map(m => m.beforeEnter),
  // async components
  resolveAsyncComponents(activated)
)
```

1. 首先在失活组件调用离开守卫
2. 调用全局的`beforeEach`守卫
3. 在重用组件中调用`beforeRouteUpdate`守卫
4. 在路由配置中调用`beforeEnter`守卫
5. 解析异步路由组件

这也和[Vue Router官网](https://router.vuejs.org/zh/guide/advanced/navigation-guards.html#组件内的守卫)中关于路由导航解析流程一致。我们一个个来看一下：

#### 2.3.1 离开守卫 extractLeaveGuards

定义在`src/history/base.js`中，执行了`extractGuards`方法：

这个方法接收 4 个参数

1. deactivated：失活组件 record
2. beforeRouteLeave：导航守卫名称
3. bindGuard：绑定组件实例到上下文函数
4. reverse：是否需要反转组件数组（反转是由于激活时从父到子，失活时应该从子到父）

```js
function extractLeaveGuards (deactivated: Array<RouteRecord>): Array<?Function> {
  return extractGuards(deactivated, 'beforeRouteLeave', bindGuard, true)
}
```

`extractGuards 和 bindGuard`方法如下：

```js
function bindGuard (guard: NavigationGuard, instance: ?_Vue): ?NavigationGuard {
  if (instance) {
    return function boundRouteGuard () {
      return guard.apply(instance, arguments)
    }
  }
}

function extractGuards (
  records: Array<RouteRecord>,
  name: string,
  bind: Function,
  reverse?: boolean
): Array<?Function> {
  const guards = flatMapComponents(records, (def, instance, match, key) => {
    const guard = extractGuard(def, name)
    if (guard) {
      return Array.isArray(guard)
        ? guard.map(guard => bind(guard, instance, match, key))
        : bind(guard, instance, match, key)
    }
  })
  return flatten(reverse ? guards.reverse() : guards)
}

function extractGuard (
  def: Object | Function,
  key: string
): NavigationGuard | Array<NavigationGuard> {
  if (typeof def !== 'function') {
    // extend now so that global mixins are applied.
    def = _Vue.extend(def)
  }
  return def.options[key]
}

// src/util/resolve-components.js 里的 flatMapComponents 和 flatten 函数
export function flatMapComponents (
  matched: Array<RouteRecord>,
  fn: Function
): Array<?Function> {
  return flatten(matched.map(m => {
    return Object.keys(m.components).map(key => fn(
      m.components[key],
      m.instances[key],
      m, key
    ))
  }))
}

export function flatten (arr: Array<any>): Array<any> {
  return Array.prototype.concat.apply([], arr)
}
```

`extractGuards`函数执行了`flatMapComponents`得到`guards`数组，这个数组是原来的`records`数组经过`map`返回的，再通过`flatten`函数拍平成一维数组。

record 里有一个 components 属性，内容可以参考 Vue Router 的[命名视图](https://router.vuejs.org/zh/guide/essentials/named-views.html)，只是我们平时写的时候写的普通版 component 而已。

![image-20191121161550559](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100559.png)

获得所有的属性名组成的数组，然后遍历返回 `fn`函数执行的结果，fn 参数拿到对应的 component 以及 组件实例 instance 供`bindGuard`绑定上下文（这个之后再介绍，// todo），fn 函数则是`extraGurad`中调用`flatMapComponents`中的第二个参数：

```js
(def, instance, match, key) => {
    const guard = extractGuard(def, name)
    if (guard) {
      return Array.isArray(guard)
        ? guard.map(guard => bind(guard, instance, match, key))
        : bind(guard, instance, match, key)
    }
  }
```

`extractGuard`函数则负责调用`Vue.extend`创建组件以及返回组件内对应`name`的钩子函数，如果有则绑定上下文为组件实例。

最后执行`return flatten(reverse ? guards.reverse() : guards)`语句，得到一个所有组件上的`beforeRouteLeave`函数数组也就是`guards`。

由于现在是失活，所以 reverse 是 true 需要翻转。此处我们没有定义，所以执行结果就是一个空数组了，如果不是空就是在`iterator`里去执行`guards`里的函数。

#### 2.3.2 全局 beforeEach 守卫

当`runQueue`执行完`extractLeaveGuards(deactivated)`后，接着就是通过`iterator`执行`this.router.beforeHooks`数组里的方法了。

beforeHooks 是定义在`src/index.js`里的，通过`beforeEach`方法注册：

```js
beforeEach (fn: Function): Function {
  return registerHook(this.beforeHooks, fn)
}

function registerHook (list: Array<any>, fn: Function): Function {
  list.push(fn)
  return () => {
    const i = list.indexOf(fn)
    if (i > -1) list.splice(i, 1)
  }
}
```

再来看看我们定义在`main.js`里的全局守卫：

```js
router.beforeEach((to, from, next) => {
  console.log('router beforeEach');
  next();
});
```

那么在`registerHook`时，如果我们多次定义了同一个，会移除已存在的。那么此时我们的 beforeHoos 为：

```js
this,beforeHooks = [
  (to, from, next) => {
    console.log('router beforeEach');
    next();
  },
]
```

而`iterator`则会执行这个函数：

```js
hook(route, current, (to: any) => {
  if (to === false || isError(to)) {
    // next(false) -> abort navigation, ensure current URL
    this.ensureURL(true)
    abort(to)
  } else if (
    typeof to === 'string' ||
    (typeof to === 'object' &&
     (typeof to.path === 'string' || typeof to.name === 'string'))
  ) {
    // next('/') or next({ path: '/' }) -> redirect
    abort()
    if (typeof to === 'object' && to.replace) {
      this.replace(to)
    } else {
      this.push(to)
    }
  } else {
    // confirm transition and pass on the value
    next(to)
  }
})
```

由于我们什么也没干，就输出了一句 log 接着执行 next()，故直接跳过了里面的步骤，如果我们在调用 next 函数时传递一些参数（例如 false），就会触发 if 里面的逻辑判断（拒绝跳转或者是跳转到某些路径）。

#### 2.3.3 重用组件内的 beforeRouteUpdate 守卫

紧接在全局`beforeEach`后，在重用的组件内调用`beforeRouteUpdate`守卫：

```js
function extractUpdateHooks (updated: Array<RouteRecord>): Array<?Function> {
  return extractGuards(updated, 'beforeRouteUpdate', bindGuard)
}
```

这里和离开守卫类似，只不过不需要反转最后的`guards`了。

#### 2.3.4 路由配置项里的 beforeEnter

`activated.map(m => m.beforeEnter)`，activated 也就是激活的 record，通过`map`遍历得到一个`beforeEnter`组成的函数数组，在`iterator`里执行。

#### 2.3.5 处理异步组件 resolveAsyncComponents

Vue Router 是支持异步组件的，而处理的关键部分就在这里，这也是为什么需要`queue`、`iterator`、`runQueue`的原因，改方法定义在`src/util/resolve-components.js`：

```js
export function resolveAsyncComponents (matched: Array<RouteRecord>): Function {
  return (to, from, next) => {
    let hasAsync = false
    let pending = 0
    let error = null

    flatMapComponents(matched, (def, _, match, key) => {
      // if it's a function and doesn't have cid attached,
      // assume it's an async component resolve function.
      // we are not using Vue's default async resolving mechanism because
      // we want to halt the navigation until the incoming component has been
      // resolved.
      if (typeof def === 'function' && def.cid === undefined) {
        hasAsync = true
        pending++

        const resolve = once(resolvedDef => {
          if (isESModule(resolvedDef)) {
            resolvedDef = resolvedDef.default
          }
          // save resolved on async factory in case it's used elsewhere
          def.resolved = typeof resolvedDef === 'function'
            ? resolvedDef
            : _Vue.extend(resolvedDef)
          match.components[key] = resolvedDef
          pending--
          if (pending <= 0) {
            next()
          }
        })

        const reject = once(reason => {
          const msg = `Failed to resolve async component ${key}: ${reason}`
          process.env.NODE_ENV !== 'production' && warn(false, msg)
          if (!error) {
            error = isError(reason)
              ? reason
              : new Error(msg)
            next(error)
          }
        })

        let res
        try {
          res = def(resolve, reject)
        } catch (e) {
          reject(e)
        }
        if (res) {
          if (typeof res.then === 'function') {
            res.then(resolve, reject)
          } else {
            // new syntax in Vue 2.3
            const comp = res.component
            if (comp && typeof comp.then === 'function') {
              comp.then(resolve, reject)
            }
          }
        }
      }
    })

    if (!hasAsync) next()
  }
}
```

他也是返回一个 guard 类型的函数，`flatMapComponents`函数我们在前面也介绍过，这里就带过了。主要介绍回调的匿名函数内部内容。

def 参数主要还是我们在定义 route 时写的 component，一般我们都是引入一个写好的组件，有时也会通过`import() `方法来动态引入组件。

此时匿名函数 typeof 判断结果，不是函数或 cid 已经存在，则代表这个组件是静态加载或者已经引入过，则直接进入`next()`。

如果满足条件后，则会将标识`hasAsync`设置 true，然后定义`resolve`和`reject`函数，对不同结果进行对应的操作。然后执行我们定义在 component 里的方法，得到结果，通过`then`方法后`resolve`或者`reject`，这样就加载完异步组件，在`iterator`中调用`resolve`或`reject`中的`next()`到下一个`step`了。

这样一来，runQueue 完，我们拿到了这一次跳转所有激活的组件了，最后执行`runQueue`里的匿名回调 cb：

### 2.4 runQueue 里的匿名回调

```js
() => {
      const postEnterCbs = []
      const isValid = () => this.current === route
      // wait until async components are resolved before
      // extracting in-component enter guards
      const enterGuards = extractEnterGuards(activated, postEnterCbs, isValid)
      const queue = enterGuards.concat(this.router.resolveHooks)
      runQueue(queue, iterator, () => {
        if (this.pending !== route) {
          return abort()
        }
        this.pending = null
        onComplete(route)
        if (this.router.app) {
          this.router.app.$nextTick(() => {
            postEnterCbs.forEach(cb => {
              cb()
            })
          })
        }
      })
    }
```

定义了`postEnterCbs`一个回调函数的数组和一个用来判断路径是否有效的标志`isValid`。接着调用`extractEnterGuards`函数，传入三个参数，来看看这个函数：

```js
function extractEnterGuards (
  activated: Array<RouteRecord>,
  cbs: Array<Function>,
  isValid: () => boolean
): Array<?Function> {
  return extractGuards(
    activated,
    'beforeRouteEnter',
    (guard, _, match, key) => {
      return bindEnterGuard(guard, match, key, cbs, isValid)
    }
  )
}

function bindEnterGuard (
  guard: NavigationGuard,
  match: RouteRecord,
  key: string,
  cbs: Array<Function>,
  isValid: () => boolean
): NavigationGuard {
  return function routeEnterGuard (to, from, next) {
    return guard(to, from, cb => {
      if (typeof cb === 'function') {
        cbs.push(() => {
          // #750
          // if a router-view is wrapped with an out-in transition,
          // the instance may not have been registered at this time.
          // we will need to poll for registration until current route
          // is no longer valid.
          poll(cb, match.instances, key, isValid)
        })
      }
      next(cb)
    })
  }
}
```

`extractGuards`函数我们之前有提到过，这里传入已激活的组件，beforeRouteEnter 钩子名，以及最后一个有点不同的 bind 函数。

+ guard：我们在`extractGuards`函数里得到的 guards 数组里的钩子函数
+ _：instance 实例，由于我们现在还没有进入到组件，也就无法获得对应的实例
+ match：当前路由匹配到的 record
+ key：component 的属性名

返回`bindEnterGuard`函数执行结果。

`bindEnterGuard`函数返回一个叫做`routeEnterGuard`的函数，也就是一个 guard 形式的函数，内部执行参数`guard`方法，并返回。

`guard`方法会判断 cb 是否为函数，如果是就存入一个调用`poll`方法的方法，将 cb 等参数传入。否则则直接执行 next，传入 cb。这个 cb 也就是我们写在`next((vm) => console.log(vm))` 里的回调。

cbs push 的这个函数里执行了`poll`函数：

```js
function poll (
  cb: any, // somehow flow cannot infer this is a function
  instances: Object,
  key: string,
  isValid: () => boolean
) {
  if (
    instances[key] &&
    !instances[key]._isBeingDestroyed // do not reuse being destroyed instance
  ) {
    cb(instances[key])
  } else if (isValid()) {
    setTimeout(() => {
      poll(cb, instances, key, isValid)
    }, 16)
  }
}
```

这个函数是一个轮训，一直去尝试获取组件实例，一旦获得，则执行 cb 方法，参数是对应组件的实例。

**这里解释了为什么 beforeEnter 里不能获取 this，但是可以通过 next() 函数的回调里拿到 this 的原因。**

通过`extractEnterGuards`函数执行，我们拿到了所有的 beforeEnter 钩子然后与 resolve 钩子合并得到一个 queue，resolve 钩子类似 beforeEach 钩子，这里可以参考那部分。

接着 runQueue:

```js
runQueue(queue, iterator, () => {
  if (this.pending !== route) {
    return abort()
  }
  this.pending = null
  onComplete(route)
  if (this.router.app) {
    this.router.app.$nextTick(() => {
      postEnterCbs.forEach(cb => {
        cb()
      })
    })
  }
})
```

当正跳转的路由不是当前传入的路由则 abort 掉，pending 路由设置为 null，执行 onComplete 方法，也就是`transitionTo`函数的第二个参数，最后在`nextTick`中将前面 `push`的 cb 进行执行。

我们来看看点击跳转时触发的 hashHistory`push`方法中的代码：

```js
// confirmTransition 里的 onComplete
() => {
  this.updateRoute(route)
  onComplete && onComplete(route)
  this.ensureURL()

  // fire ready cbs once
  if (!this.ready) {
    this.ready = true
    this.readyCbs.forEach(cb => {
      cb(route)
    })
  }
}

// transitionTo 里的 onComplete
route => {
  pushHash(route.fullPath)
  handleScroll(this.router, route, fromRoute, false)
  onComplete && onComplete(route)
}
```

`confirmTransition`里的`onComplete`会执行`updateRoute`函数更新路由：

```js
updateRoute (route: Route) {
  const prev = this.current
  this.current = route
  this.cb && this.cb(route)
  this.router.afterHooks.forEach(hook => {
    hook && hook(route, prev)
  })
}

// src/index.js
afterEach (fn: Function): Function {
  return registerHook(this.afterHooks, fn)
}
```

这里就是同`beforeEach`、`beforeResolve`一样，注册事件钩子，这就是调用全局的`afterEach`钩子。

`transitionTo`里的`onComplete`前两个函数就是替换 url 里的路由以及处理滚动，点击`router-link`，onComplete 是空函数也就没有什么操作。

到这里就完成了

7. 在被激活的组件里调用 `beforeRouteEnter`。
8. 调用全局的 `beforeResolve` 守卫 (2.5+)。
9. 导航被确认。
10. 调用全局的 `afterEach` 钩子。
11. 触发 DOM 更新。
12. 用创建好的实例调用 `beforeRouteEnter` 守卫中传给 `next` 的回调函数。

## 总结

回顾梳理一下这一篇的主要内容：

1. 我们点击了 router-link 来跳转路由
2. router-link 中触发点击事件，从而触发 Router 的 push 事件
3. push 事件中调用了 transitionTo 方法
4. tansitionTo 主要执行 comfirmTransition 方法
5. comfirmTransition 方法内首先通过 resolveQueue 方法拿到需要更新的组件、激活的组件以及废弃的组件对象
6. 再将不同的钩子函数以及异步组件加载存入到数组 queue
7. 通过 runQueue 方法顺序执行 queue 里的函数
8. 整个执行完后调用 runQueue 中的回调执行 resolve、after 钩子
9. 触发DOM 更新
10. 将创建好的实例传递到 beforeRouterEnter 的回调里

到这里我们对 Vue Router 的基本工作原理已经分析完毕了，最重要的概念是`Router`、`Route`、`Location`、`Record`几个对象，尤其是`Record`，涉及到匹配的路由地方的内容都是由这个对象来决定。

