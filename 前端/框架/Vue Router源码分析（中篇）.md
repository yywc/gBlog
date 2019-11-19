## 前言

承接上一篇，我们知道了 Router 初始化过程做了一些什么，而且从`http://localhost:8080`到`http://localhost:8080/#/`的过程，那么我们在输入一个路径时，Router 是如何找到对应的组件来渲染的呢，这一篇主要就是介绍这个匹配过程。

我们以`http://localhost:8080/#/foo/child/10000`这个路由刷新为例。

## 1. createMatcher 函数

在初始化 Router 的时候，matcher 相关的我们跳过了：`this.matcher = createMatcher(options.routes || [], this)`，matcher 由`createMatcher`方法返回,它定义在`src/create-matcher.js`里：

```js
export function createMatcher (
  routes: Array<RouteConfig>,
  router: VueRouter
): Matcher {
  const { pathList, pathMap, nameMap } = createRouteMap(routes)
  
  function addRoutes (routes) {
    createRouteMap(routes, pathList, pathMap, nameMap)
  }
  
  function match (
    raw: RawLocation,
    currentRoute?: Route,
    redirectedFrom?: Location
  ): Route { /*...*/ }
  
  function redirect (
    record: RouteRecord,
    location: Location
  ): Route { /*...*/ }
  
  function alias (
    record: RouteRecord,
    location: Location,
    matchAs: string
  ): Route { /*...*/ }
  
  function _createRoute (
    record: ?RouteRecord,
    location: Location,
    redirectedFrom?: Location
  ): Route { /*...*/ }
  
  return {
    match,
    addRoutes
  }
}
```

`createMatcher`接收两个参数，第一个是我们在初始化时传入的 routes 数组，第二个则是 Router 对象，它实际上就是进行了一系列的操作，返回了一个对象，里面包含`match`方法和`addRoutes`方法。

那么首先从上往下看下，也就是执行了第一行的代码：`const { pathList, pathMap, nameMap } = createRouteMap(routes)`通过执行`createRouteMap`方法返回了三个值分别是：

+ pathList：路由路径列表，我们自己编写的例子中就是：['/foo/child/:id', 'foo', 'bar']
+ pathMap：路由映射对象，key 是 path，value 是 RouteRecord 对象
+ nameMap：名称映射表，key 是 name，value 是 RouteRecord 对象

## 2. createRouteMap 函数

该方法定义在`src/create-route-map.js`中（去除部分不影响主逻辑的代码）：

```js
export function createRouteMap (
  routes: Array<RouteConfig>,
  oldPathList?: Array<string>,
  oldPathMap?: Dictionary<RouteRecord>,
  oldNameMap?: Dictionary<RouteRecord>
): {
  pathList: Array<string>,
  pathMap: Dictionary<RouteRecord>,
  nameMap: Dictionary<RouteRecord>
} {
  // the path list is used to control path matching priority
  const pathList: Array<string> = oldPathList || []
  // $flow-disable-line
  const pathMap: Dictionary<RouteRecord> = oldPathMap || Object.create(null)
  // $flow-disable-line
  const nameMap: Dictionary<RouteRecord> = oldNameMap || Object.create(null)

  // routes = [{ path: '/foo', ... }, { path: '/bar', ... }]
  routes.forEach(route => {
    // 以 { path: '/foo', ... } 分析
    addRouteRecord(pathList, pathMap, nameMap, route)
  })

  // ensure wildcard routes are always at the end
  for (let i = 0, l = pathList.length; i < l; i++) {
    if (pathList[i] === '*') {
      pathList.push(pathList.splice(i, 1)[0])
      l--
      i--
    }
  }

  return {
    pathList,
    pathMap,
    nameMap
  }
}
```

首次进入`createRouteMap`方法，除了 routes，其余参数都是 undefined，故 pathList = []，pathMap = {}，nameMap = {}，然后遍历 routes，执行`addRouteRecord`，之后判断 path 中是否有通配符，如果有则放到最后去，最后返回经由`addRouteRecord`处理过的 pathList、pathMap、nameMap 对象。

## 3. addRouteRecord 函数

下面看一下`addRouteRecord`方法，由于这个方法有点长，我们去除一些不影响主要逻辑的代码和警告判断。

```js
function addRouteRecord (
  pathList: Array<string>, // []
  pathMap: Dictionary<RouteRecord>, // {}
  nameMap: Dictionary<RouteRecord>, // {}
  route: RouteConfig, // { path: '/foo', ... }
  parent?: RouteRecord, // undefined
  matchAs?: string // undefined
) {
  // 从 route 对象中解构出 path 和 name
  // route: {
  //   path: '/foo',
  //   name: 'Foo',
  //   component: Foo,
  //   meta: { permission: true },
  //   children: [
  //     {
  //       path: 'child/:id',
  //       name: 'Child',
  //       component: Child,
  //     },
  //   ],
  // },
  const { path, name } = route // path = '/foo', name = 'Foo'

  // ('/foo'、undefined、undefined) normalizedPath = 'foo'
  const normalizedPath = normalizePath(path, parent, pathToRegexpOptions.strict)
```

> function  normalizePath  (
>     path:  string,
>     parent?:  RouteRecord,
>     strict?:  boolean
> ):  string  {
>     if  (!strict)  path  =  path.replace(/\/\$/,  '')
>     if  (path[0]  ===  '/')  return  path  //  returrn "/foo"
>     if  (parent  ==  null)  return  path
>     return  cleanPath(\`${parent.path}/\${path}`)
>
> }

```js
  const record: RouteRecord = {
    path: normalizedPath, // path = "/foo"
    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions), // regex = {}
    components: route.components || { default: route.component }, // components = { default: Foo }
    instances: {},
    name, // name = "Foo"
    parent, // parent = undefined
    matchAs,  // matchAs = undefined
    redirect: route.redirect, // redirect = undefined
    beforeEnter: route.beforeEnter, // beforeEnter = undefined
    meta: route.meta || {}, // meta = {}
    props: // props = {}
      route.props == null
        ? {}
        : route.components
          ? route.props
          : { default: route.props }
  }

  // 递归遍历 children 同样的操作
  if (route.children) {
    route.children.forEach(child => {
      const childMatchAs = matchAs
        ? cleanPath(`${matchAs}/${child.path}`)
        : undefined
      addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs)
    })
  }

  // 存入 record 的 path 和将 record 存入 pathMap、nameMap，方便后续取出
  if (!pathMap[record.path]) {
    pathList.push(record.path)
    pathMap[record.path] = record
  }

  if (name) {
    if (!nameMap[name]) {
      nameMap[name] = record
    } else if (process.env.NODE_ENV !== 'production' && !matchAs) {
      warn(
        false,
        `Duplicate named routes definition: ` +
          `{ name: "${name}", path: "${record.path}" }`
      )
    }
  }
}
```

接下来需要重点介绍一下 RouteRecord 对象，这个对象其实长得和我们在 routes 数组里写的对象很相似，有 path、name、component 等属性，可以理解成一个“仓库”保存了我们路由相关的一些属性信息。

然后判断 RouteRecord 有无 children，在我们的例子中是有的，所有会再次递归执行`addRouteRecord`方法，直到遍历完所有的 children。

接下来判断 pathMap 对象时是否含有 key 为 path 的值，没有则在 pathList 数组 push 当前 path，在 pathMap 对象中建立 path 与 RouteRecord 的映射。

最后是判断有无 name 属性，建立 name 与 record 的映射。

这样我们就可以通过 path 或者 name 正确地找到对应 RouteRecord。

通过执行`addRouteRecord`后，我们就获得了一个保存有数据的 pathList、pathMap、nameMap 了。

理一理之前的逻辑：

+ 我们首先是调用`createMatcher`创建 matcher
+ 在`createMatcher`方法中执行了`createRouteMap`方法得到三个值也就是 pathList、pathMap、nameMap
+ 最后返回了一个`match`函数和`addRoutes`函数

## 4. match 函数

那我们现在来分析一下`match`函数又是做了些什么。

```js
function match (
  raw: RawLocation, // raw = "/foo/child/10000"
  currentRoute?: Route, // currentRoute = { name: null, path: "/", hash: "", ... }
  redirectedFrom?: Location // undefined
): Route {
  // location: {
  //   hash: ""
  //   path: "/foo/child/10000"
  //   query: {}
  // }
  const location = normalizeLocation(raw, currentRoute, false, router)
  const { name } = location // name = undefined

  if (name) {
    const record = nameMap[name]
    if (process.env.NODE_ENV !== 'production') {
      warn(record, `Route with name '${name}' does not exist`)
    }
    if (!record) return _createRoute(null, location)
    const paramNames = record.regex.keys
      .filter(key => !key.optional)
      .map(key => key.name)

    if (typeof location.params !== 'object') {
      location.params = {}
    }

    if (currentRoute && typeof currentRoute.params === 'object') {
      for (const key in currentRoute.params) {
        if (!(key in location.params) && paramNames.indexOf(key) > -1) {
          location.params[key] = currentRoute.params[key]
        }
      }
    }

    location.path = fillParams(record.path, location.params, `named route "${name}"`)
    return _createRoute(record, location, redirectedFrom)
  } else if (location.path) {
    location.params = {}
    for (let i = 0; i < pathList.length; i++) {
      const path = pathList[i]
      const record = pathMap[path]
      // 如果匹配到了路径则返回以该 record、location 里的相关属性创建的路径
      if (matchRoute(record.regex, location.path, location.params)) {
        return _createRoute(record, location, redirectedFrom)
      }
    }
  }
  // no match
  return _createRoute(null, location)
}
```

这里有两个对象 Rawlocation 和 Location，我们简单了解一下：

```ts
declare type Location = {
  _normalized?: boolean;
  name?: string;
  path?: string;
  hash?: string;
  query?: Dictionary<string>;
  params?: Dictionary<string>;
  append?: boolean;
  replace?: boolean;
}

declare type RawLocation = string | Location
```

Location 对象就是保存了路径上的一些属性值，而 RawLocation 就是一个字符串或者 Location 对象。

接着函数体第一步就执行了一个`normalizeLocation`函数，这个函数主要是将不同格式的 raw 格式化成 Location 对象。里面逻辑挺复杂而且我们不太需要去理清这些，但是我们可以打开`test/unit/specs/location.spec.js`单元测试文件看看他会做一些什么操作：

```js
describe('normalizeLocation', () => {
  it('string', () => {
    const loc = normalizeLocation('/abc?foo=bar&baz=qux#hello')
    expect(loc._normalized).toBe(true)
    expect(loc.path).toBe('/abc')
    expect(loc.hash).toBe('#hello')
    expect(JSON.stringify(loc.query)).toBe(JSON.stringify({
      foo: 'bar',
      baz: 'qux'
    }))
  })
}
```

当我们的路径是`/abc?foo=bar&baz=qux#hello`的时候，`normalizeLocation`函数会返回一个这样一个对象：

```js
location = {
  _normalized: true,
  path: '/abc',
  hash: 'hello',
  query: {
    foo: 'bar',
    baz: 'qux',
  },
}
```

这样一来我们就需要去了解`normalizeLocation`内部实现，而知道他做了些什么了，这也是阅读源码时的一种“偷懒”的方法。

继续我们的主线，拿到 location 后，我们首先从中取出了 name，如果有 name，我们直接从 nameMap 中取到 record 对象，而当这个对象不存在，我们会调用`_createRoute`方法来创建一条空路径返回。

```js
function _createRoute (
  record: ?RouteRecord,  // record = { path: "/foo/child/:id", regex: xxxx, components: xxx, ... }
  location: Location, // location = { _normalized: true, path: "/foo/child/10000", xxx }
  redirectedFrom?: Location // undefined
): Route {
  if (record && record.redirect) {
    return redirect(record, redirectedFrom || location)
  }
  if (record && record.matchAs) {
    return alias(record, location, record.matchAs)
  }
  // redirect、matchAs 都不存在，所以会执行 createRoute 方法
  return createRoute(record, location, redirectedFrom, router)
}

// createRoute 定义在 src/util/route.js 中
export function createRoute (
  record: ?RouteRecord,
  location: Location,
  redirectedFrom?: ?Location,
  router?: VueRouter
): Route {
  // 字符串化查询参数
  const stringifyQuery = router && router.options.stringifyQuery // stringifyQuery = undefined

  // 获取查询参数
  let query: any = location.query || {} // query = {}
  // 拷贝 query
  try {
    query = clone(query) // query = {}
  } catch (e) {}

  // 创建 Route 对象
  const route: Route = {
    name: location.name || (record && record.name), // name = "Child"
    meta: (record && record.meta) || {}, // meta = {}
    path: location.path || '/', // path = "/foo/child/10000"
    hash: location.hash || '',  // hash = ""
    query, // query = {}
    params: location.params || {}, // params = { id: 10000 }
    fullPath: getFullPath(location, stringifyQuery), // fullPath = "/foo/child/10000"
    // matched = [{ path: '/foo', ... }, { path: '/foo/child/:id', ... }]
    matched: record ? formatMatch(record) : []
  }
  if (redirectedFrom) {
    route.redirectedFrom = getFullPath(redirectedFrom, stringifyQuery)
  }
  // 返回 route
  return Object.freeze(route)
}
```

到这里`match`方法的工作就算做完了，我们回顾一下，他主要是做了以下三件事情：

1. 格式化路由和当前路由为 Location 对象
2. 根据 location 中的 name 或者 path 来找到 record
3. 通过 record 和 location 创建 Route 对象

## 5. addRoutes 函数

接下来的`addRoutes`方法我们其实相当于分析过了：

```js
function addRoutes (routes) {
  createRouteMap(routes, pathList, pathMap, nameMap)
}
```

他就是提供了一个供外部动态添加路径的方法，然后传入新的 routes 以及 pathList，pathMap，nameMap 这些值，执行`createRouteMap`时会就会从原有的基础上去修改他们了，保证我们自己添加的路径也能正确地被初始化。

## 总结

这样下来，我们终于分析完了 matcher 相关的逻辑，可以看到，内部的逻辑是非常复杂的，我们要想理清楚，有些逻辑或者辅助函数大可不必去细究，只需要知道他是干嘛的就行了，像前面提到的通过看单元测试的方法就是一个很好的切入点。

我们分析`matcher`是为了知道我们第一次进入时如何正确地渲染到我们定义的组件，同时也是为了分析切换路由时 Vue Router 做了哪些工作打个基础。