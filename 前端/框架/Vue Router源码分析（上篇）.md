## 前言

我们在使用 Vue 相关生态的时候，避免不了的会使用到 Vue Router，而关于 Vue Router 背后如何帮助我们管理路由，渲染页面，跳转路径，却知之甚少，这一篇主要从大部分同学使用到的一些场景来分析它的原理。

## 1.  开始写一个 Router

我们以 Vue 单页应用为例:

`main.js`

```js
import Vue from 'vue';
import App from './App';
import router from './router';

Vue.config.productionTip = false;

router.beforeEach((to, from, next) => {
  console.log('router beforeEach');
  next();
});

router.beforeResolve((to, from, next) => {
  console.log('router beforeResolve');
  next();
});

router.afterEach((to, from) => {
  console.log('router afterEach =====', from.name);
});

/* eslint-disable no-new */
new Vue({
  el: '#app',
  router,
  template: '<App />',
  components: {
    App,
  },
});
```

`router/index.js`

```js
import Vue from 'vue';
import Router from 'vue-router';

Vue.use(Router);

const Foo = {
  template: `
  <div>
    Foo
    <div>
      <router-link to="/foo/child/10000">to-child</router-link>
      <router-view />
    </div>
  </div>`,
};

const Child = { template: '<div>Foo Child</div>' };

const Bar = { template: '<div>Bar</div>' };

export default new Router({
  routes: [
    {
      path: '/',
      name: 'Foo',
      component: Foo,
    },
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
    },
  ],
});
```

`App.vue`

```vue
<template>
  <div id="app">
    <h1>Hello Router</h1>
    <div>
      <router-link to="/foo">to foo</router-link>
      <router-link to="/bar">to bar</router-link>
    </div>
    <router-view />
  </div>
</template>

<script>

export default {
  name: 'App',
};
</script>

<style lang="css">
#app {
  text-align: center;
  margin-top: 200px;
}

#app > div {
  margin-bottom: 20px;
}

#app > div > a {
  margin: 10px;
}
</style>
```

运行项目时如下：

![image-20191108101151104](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-08-021655.jpg)

下面开始一步步分析 Vue Router 的内容。

## 2. import Router from 'vue-router'

这一步主要是从`src/index.js`中引入了 VueRouter 类，包含一个`install`静态方法和`version`版本号。类中定义了构造器和一些方法。

## 3. Vue.use(Router)

这里先提前介绍两个对象：

+ Router：VueRouter 路由对象，提供一系列方法操作路由例如`this.$router.push`等等；
+ Route：路径对象，提供路径的一些参数，例如`this.$route.query`等等；

`Vue.use()`方法给插件提供了一个注入到 Vue 中的方法，它会调用插件的`install`方法，该方法在`src/install.js`中：

```js
import View from './components/view'
import Link from './components/link'

export let _Vue  // 导出 Vue 对象供其他地方使用

export function install (Vue) {
  // instanlled 方法和 _Vue 对象联合判断
  // 已经 install 了则返回，避免重复安装
  if (install.installed && _Vue === Vue) return

  install.installed = true  // installed 标志设置为已安装状态

  _Vue = Vue // 存储 Vue 对象

  const isDef = v => v !== undefined // 判别变量是否被定义的函数

  // 注册实例方法：取父节点的 data 中的 registerRouteInstance 方法进行注册，该方法后续会提到
  // 这里只需要知道 registerInstance callVal 不为空是注册 Route，为空是注销。
  const registerInstance = (vm, callVal) => {
    let i = vm.$options._parentVnode
    if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
      i(vm, callVal)
    }
  }

  // 混入到 Vue
  Vue.mixin({
    beforeCreate () {

      // 这里是我们在 main.js 中传入的 router，判断有无
      if (isDef(this.$options.router)) {
        this._routerRoot = this  // 根 Vue 实例
        this._router = this.$options.router  // 当前 router 对象
        this._router.init(this)  // 初始化 router

        // definedReact 将 this._route 设置成响应式对象
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      } else {
        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this  // 取根 Vue 实例
      }
      registerInstance(this, this)  // 注册 Route 实例
    },
    destroyed () {
      registerInstance(this)  // 销毁 Route 实例
    }
  })

  // 将 router 实例挂载到 $router 上，由是我们可以在 vue 文件中通过 this.$router 访问到路有对象
  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this._routerRoot._router }
  })

  // 将 route 实例挂载到 $route 上，同上
  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this._routerRoot._route }
  })

  // 注册的 RouterView 和 RouterLink 组件，先过。
  Vue.component('RouterView', View)
  Vue.component('RouterLink', Link)

  // 挂载 beforeRouteEnter、beforeRouteLeave、beforeRouteUpdate 方法到 Vue 上
  const strats = Vue.config.optionMergeStrategies
  // use the same hook merging strategy for route hooks
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
}

```

接着上面`this._router.init(this)`逻辑，`init`方法定义在`src/index.js`中，是 VueRouter 的实例方法。

```js
init (app: any /* Vue component instance */) {
  process.env.NODE_ENV !== 'production' && assert(
    install.installed,
    `not installed. Make sure to call \`Vue.use(VueRouter)\` ` +
    `before creating root instance.`
  )

  this.apps.push(app)

  // set up app destroyed handler
  // https://github.com/vuejs/vue-router/issues/2639
  app.$once('hook:destroyed', () => {
    // clean out app from this.apps array once destroyed
    const index = this.apps.indexOf(app)
    if (index > -1) this.apps.splice(index, 1)
    // ensure we still have a main app or null if no apps
    // we do not release the router so it can be reused
    if (this.app === app) this.app = this.apps[0] || null
  })

  // main app previously initialized
  // return as we don't need to set up new history listener
  if (this.app) {
    return
  }

  this.app = app

  const history = this.history

  if (history instanceof HTML5History) {
    history.transitionTo(history.getCurrentLocation())
  } else if (history instanceof HashHistory) {
    const setupHashListener = () => {
      history.setupListeners()
    }
    history.transitionTo(
      history.getCurrentLocation(),
      setupHashListener,
      setupHashListener
    )
  }

  history.listen(route => {
    this.apps.forEach((app) => {
      app._route = route
    })
  })
}
```

先判断了是否安装插件，然后将 vm 存入到 apps 数组，当 vm 销毁，也要将 vm 从 apps 中移除，避免内存溢出。

接着判断 this.app 是否存在，若无则初始化，然后获取 history 对象，这个对象在下面 new Router() 初始化 Router 对象时介绍。由于我们常用 hashHistory 模式，所以就从这部分分析。

在 else if 里定义了一个 setupHashListener 函数，它的作用就是设置 history 的监听（滚动行为）；

然后调用`transitionTo`方法跳转路由，传入 currentLocation 以及 setupListeners 函数；

最后调用了 history.listen 方法将匿名函数挂载到 history 的 cb 属性上。

## 4. 初始化 Router 对象

当我们写完 Vue.use() 后，接下来就是初始化 Router 对象了：

```js
const router = new Router({
  routes: [
    {
      path: '/',
      component: HelloWorld
    },
  ],
});
```

这个参数对象还支持一些其他的属性：mode—默认采用 hash 模式、scrollBehavior—路由切换时的滚动效果、base—基础路径等等，更多可以查看[官网构建选项]([https://router.vuejs.org/zh/api/#router-%E6%9E%84%E5%BB%BA%E9%80%89%E9%A1%B9](https://router.vuejs.org/zh/api/#router-构建选项))。

Router 类定义在`src/index.js`下，简单来看 constructor 部分：

```js
constructor (options: RouterOptions = {}) {
    this.app = null // 当前 vm 实例
    this.apps = [] // vm 数组
    this.options = options // 配置项
    this.beforeHooks = [] // 钩子函数执行前的导航守卫函数数组
    this.resolveHooks = [] // 钩子函数执行时的导航守卫函数数组
    this.afterHooks = [] // 钩子函数执行后的导航钩子函数数组
    this.matcher = createMatcher(options.routes || [], this) // * 路径匹配器，后续介绍

    let mode = options.mode || 'hash' // 默认是 hash
    // 下面是对 history 模式支持情况的判断，不支持则回退到 hash 模式
    this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
    if (this.fallback) {
      mode = 'hash'
    }
    if (!inBrowser) {
      mode = 'abstract'
    }
    this.mode = mode

    // 初始化 history 对象
    switch (mode) {
      case 'history':
        this.history = new HTML5History(this, options.base)
        break
      case 'hash':
        this.history = new HashHistory(this, options.base, this.fallback)
        break
      case 'abstract':
        this.history = new AbstractHistory(this, options.base)
        break
      default:
        if (process.env.NODE_ENV !== 'production') {
          assert(false, `invalid mode: ${mode}`)
        }
    }
  }
```

我们来看看 HashHistory 对象，`src/history/hash.js`（constructor部分代码）

```js
export class HashHistory extends History {
  constructor (router: Router, base: ?string, fallback: boolean) {
    // 调用父类构造器
    super(router, base)
    //
    if (fallback && checkFallback(this.base)) {
      return
    }
    ensureSlash()
  }
}
```

由于一般没有定义 fallback，会进入到 ensureSlash 函数：

```js
function ensureSlash (): boolean {
  const path = getHash()
  if (path.charAt(0) === '/') {
    return true
  }
  replaceHash('/' + path)
  return false
}

export function getHash (): string {
  // We can't use window.location.hash here because it's not
  // consistent across browsers - Firefox will pre-decode it!
  let href = window.location.href
  const index = href.indexOf('#')
  // empty path
  if (index < 0) return ''

  href = href.slice(index + 1)
  // decode the hash but not the search or hash
  // as search(query) is already decoded
  // https://github.com/vuejs/vue-router/issues/2708
  const searchIndex = href.indexOf('?')
  if (searchIndex < 0) {
    const hashIndex = href.indexOf('#')
    if (hashIndex > -1) {
      href = decodeURI(href.slice(0, hashIndex)) + href.slice(hashIndex)
    } else href = decodeURI(href)
  } else {
    if (searchIndex > -1) {
      href = decodeURI(href.slice(0, searchIndex)) + href.slice(searchIndex)
    }
  }

  return href
}
```

super，执行父类构造器。父类构造器定义在`src/history/base.js`中。

```js
constructor (router: Router, base: ?string) {
  this.router = router
  this.base = normalizeBase(base) // this.base = ""
  // start with a route object that stands for "nowhere"
  this.current = START
  this.pending = null
  this.ready = false
  this.readyCbs = []
  this.readyErrorCbs = []
  this.errorCbs = []
}

function normalizeBase (base: ?string): string {
  if (!base) {
    if (inBrowser) {
      // respect <base> tag
      const baseEl = document.querySelector('base') // baseEl = null
      base = (baseEl && baseEl.getAttribute('href')) || '/' // base = "/"
      // strip full URL origin
      base = base.replace(/^https?:\/\/[^\/]+/, '') // base = "/"
    } else {
      base = '/'
    }
  }
  // make sure there's the starting slash
  if (base.charAt(0) !== '/') {
    base = '/' + base
  }
  // remove trailing slash
  return base.replace(/\/$/, '') // return ""
}

// src/util/route.js
// 创建了一条空的 Route 作为开始路径，后续会介绍 createRoute 相关。
export const START = createRoute(null, {
  path: '/'
})
```

这里主要是确定了 base 以及一条空路径。

接着 HashHistory 看，我们一般的地址是`http://localhost:8080`，会执行到 if (index < 0) 则会直接 return 空串出去，所以`ensureSlash`函数中的 path 就是空串，replaceHash 的参数是 '/'：

```js
function replaceHash (path) { // path = "/"
  if (supportsPushState) { // supportsPushState = true
    replaceState(getUrl(path))
  } else {
    window.location.replace(getUrl(path))
  }
}


// src/util/push-state.js
export const supportsPushState =
  inBrowser &&
  (function () {
    const ua = window.navigator.userAgent

    if (
      (ua.indexOf('Android 2.') !== -1 || ua.indexOf('Android 4.0') !== -1) &&
      ua.indexOf('Mobile Safari') !== -1 &&
      ua.indexOf('Chrome') === -1 &&
      ua.indexOf('Windows Phone') === -1
    ) {
      return false
    }

    return window.history && 'pushState' in window.history
  })()


// src/util/push-state.js
function getUrl (path) { // path = "/"
  const href = window.location.href // href = "http://localhost:8080/"
  const i = href.indexOf('#') // i = -1
  const base = i >= 0 ? href.slice(0, i) : href // base = "http://localhost:8080/"
  return `${base}#${path}` // return "http://localhost:8080/#/"
}


// src/util/push-state.js
export function replaceState (url?: string) { // url = "http://localhost:8080/#/"
  pushState(url, true)
}


// src/util/push-state.js
// url = "http://localhost:8080/#/", replace = true
export function pushState (url?: string, replace?: boolean) {
  saveScrollPosition()
  // try...catch the pushState call to get around Safari
  // DOM Exception 18 where it limits to 100 pushState calls
  const history = window.history
  try {
    if (replace) {
      // 调用 window.history.replaceState 更换 url，key 值是通过 performance.now() 取到的，更为精确
      history.replaceState({ key: getStateKey() }, '', url)
    } else {
      history.pushState({ key: setStateKey(genStateKey()) }, '', url)
    }
  } catch (e) {
    window.location[replace ? 'replace' : 'assign'](url)
  }
}
```

通过 pushState，我们访问的 localhost:8080 也就变成了 localhost:8080/#/。

## 总结

到这里我们就 Vue Router 的引入，初始化过程做了一个简单的了解，也知道了 \$router、\$route 是如何绑定到 Vue 上以及我们访问的`http://localhost:8080`怎么变成的`http://localhost:8080/#/`的，第二篇将来介绍路由匹配的工作原理，也就是之前我们提到过的 matcher 对象。
