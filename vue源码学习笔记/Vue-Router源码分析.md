### Vue-Router源码分析

#### 注册路由
Vue 提供了 Vue.use 的全局 API 来注册这些插件：
```
export function initUse (Vue: GlobalAPI) {
  Vue.use = function (plugin: Function | Object) {
    // installedPlugins数组保留了通过Vue.use()注册的所有插件，如果该插件已经注册过，则不会执行重新注册
    const installedPlugins = (this._installedPlugins || (this._installedPlugins = []))
    if (installedPlugins.indexOf(plugin) > -1) {
      return this
    }

    const args = toArray(arguments, 1)
    // 把Vue加入到参数的第一个位置，方便vue-router中使用到Vue，而不用单独再去import Vue了
    args.unshift(this)
    // 通常所有的插件都会有一个install方法，执行Vue.use实际就是执行了插件的install方法
    if (typeof plugin.install === 'function') {
      plugin.install.apply(plugin, args)
    } else if (typeof plugin === 'function') {
      plugin.apply(null, args)
    }
    // 插件加入installedPlugins数组，和开始判断有无这个插件对应
    installedPlugins.push(plugin)
    return this
  }
}
```
下面看下Vue-router的install方法，定义在 **src/install.js** 中：
```
export let _Vue
export function install (Vue) {
  // 如果已经安装过，不重复安装
  if (install.installed && _Vue === Vue) return
  install.installed = true

  _Vue = Vue

  const isDef = v => v !== undefined

  // ...
  
  // 在每个组件中通过Vue.mixin方法注入了beforeCreate和destroyed钩子函数
  Vue.mixin({
    beforeCreate () {
      // ...
    },
    destroyed () {
      // ...
    }
  })
   
  // ...
}
```
> Vue-Router 安装最重要的一步就是利用 Vue.mixin 去把 beforeCreate 和 destroyed 钩子函数注入到每一个组件中。
Vue.mixin方法如下定义：
```
export function initMixin (Vue: GlobalAPI) {
  Vue.mixin = function (mixin: Object) {
    // 通过合并配置项把mixin混入到Vue.options中，而每个组件的构造函数都会合并Vue.options到自身组件，所以每个组件都有了通过Vue.mixin混入的配置项
    this.options = mergeOptions(this.options, mixin)
    return this
  }
}
```
所以每个组件都有了Vue-router混入的beforeCreate 和 destroyed钩子函数。  
#### 接着看install函数
```
export let _Vue
export function install (Vue) {
  //...
  const isDef = v => v !== undefined
  Vue.mixin({
    beforeCreate () {
      // 由于是在beforeCreate钩子函数中，所以只有Vue的options上才有router，因为此时子组件还没有合并参数，子组件会走else逻辑
      if (isDef(this.$options.router)) {
        // this._routerRoot开始是Vue
        this._routerRoot = this
        // 通过new Vue传入的router实例（通过new VueRouter创建）
        this._router = this.$options.router 
        // 初始化路由，带入Vue参数
        this._router.init(this)
        // 把 this._route 变成响应式对象
        Vue.util.defineReactive(this, '_route', this._router.history.current) 
      } else {
        对子组件而言，this._routerRoot始终指向的离它最近的传入了 router 对象作为配置而实例化的父实例
        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
      }
      registerInstance(this, this)
    },
    destroyed () {
      registerInstance(this)
    }
  })

  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this._routerRoot._router }
  })

  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this._routerRoot._route }
  })
  
  // 全局注册RouterView
  Vue.component('RouterView', View)
  // 全局注册RouterLink
  Vue.component('RouterLink', Link)
  
  // 路由中钩子函数的合并策略
  const strats = Vue.config.optionMergeStrategies
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
}
```

#### 先贴路由类完整代码
```
export default class VueRouter {
  static install: () => void;
  static version: string;

  app: any;
  apps: Array<any>;
  ready: boolean;
  readyCbs: Array<Function>;
  options: RouterOptions;
  mode: string;
  history: HashHistory | HTML5History | AbstractHistory;
  matcher: Matcher;
  fallback: boolean;
  beforeHooks: Array<?NavigationGuard>;
  resolveHooks: Array<?NavigationGuard>;
  afterHooks: Array<?AfterNavigationHook>;

  constructor (options: RouterOptions = {}) {
    this.app = null
    this.apps = []
    this.options = options
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []
    this.matcher = createMatcher(options.routes || [], this)

    let mode = options.mode || 'hash'
    this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
    if (this.fallback) {
      mode = 'hash'
    }
    if (!inBrowser) {
      mode = 'abstract'
    }
    this.mode = mode

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

  match (
    raw: RawLocation,
    current?: Route,
    redirectedFrom?: Location
  ): Route {
    return this.matcher.match(raw, current, redirectedFrom)
  }

  get currentRoute (): ?Route {
    return this.history && this.history.current
  }

  init (app: any) {
    process.env.NODE_ENV !== 'production' && assert(
      install.installed,
      `not installed. Make sure to call \`Vue.use(VueRouter)\` ` +
      `before creating root instance.`
    )

    this.apps.push(app)

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

  beforeEach (fn: Function): Function {
    return registerHook(this.beforeHooks, fn)
  }

  beforeResolve (fn: Function): Function {
    return registerHook(this.resolveHooks, fn)
  }

  afterEach (fn: Function): Function {
    return registerHook(this.afterHooks, fn)
  }

  onReady (cb: Function, errorCb?: Function) {
    this.history.onReady(cb, errorCb)
  }

  onError (errorCb: Function) {
    this.history.onError(errorCb)
  }

  push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    this.history.push(location, onComplete, onAbort)
  }

  replace (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    this.history.replace(location, onComplete, onAbort)
  }

  go (n: number) {
    this.history.go(n)
  }

  back () {
    this.go(-1)
  }

  forward () {
    this.go(1)
  }

  getMatchedComponents (to?: RawLocation | Route): Array<any> {
    const route: any = to
      ? to.matched
        ? to
        : this.resolve(to).route
      : this.currentRoute
    if (!route) {
      return []
    }
    return [].concat.apply([], route.matched.map(m => {
      return Object.keys(m.components).map(key => {
        return m.components[key]
      })
    }))
  }

  resolve (
    to: RawLocation,
    current?: Route,
    append?: boolean
  ): {
    location: Location,
    route: Route,
    href: string,
    normalizedTo: Location,
    resolved: Route
  } {
    const location = normalizeLocation(
      to,
      current || this.history.current,
      append,
      this
    )
    const route = this.match(location, current)
    const fullPath = route.redirectedFrom || route.fullPath
    const base = this.history.base
    const href = createHref(base, fullPath, this.mode)
    return {
      location,
      route,
      href,
      normalizedTo: location,
      resolved: route
    }
  }

  addRoutes (routes: Array<RouteConfig>) {
    this.matcher.addRoutes(routes)
    if (this.history.current !== START) {
      this.history.transitionTo(this.history.getCurrentLocation())
    }
  }
}
```

#### 初始化路由
先来看看new VueRouter做了什么工作：
```
export default class VueRouter {
  // ...
  constructor (options: RouterOptions = {}) {
    this.app = null
    this.apps = []
    this.options = options
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []
    
    // 路由matcher，返回match和addRoutes函数
    this.matcher = createMatcher(options.routes || [], this)

    let mode = options.mode || 'hash'
    this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
    if (this.fallback) {
      mode = 'hash'
    }
    if (!inBrowser) {
      mode = 'abstract'
    }
    this.mode = mode

    switch (mode) {
      case 'history':
        // 创建history模式下的history实例
        this.history = new HTML5History(this, options.base)
        break
      case 'hash':
        // 创建hash模式下的history实例
        this.history = new HashHistory(this, options.base, this.fallback)
        break
      case 'abstract':
        // 非浏览器下的history实例
        this.history = new AbstractHistory(this, options.base)
        break
      default:
        if (process.env.NODE_ENV !== 'production') {
          assert(false, `invalid mode: ${mode}`)
        }
    }
  }
  // ...
}
```
在install中执行注入的beforeCreate钩子中执行的 **this._router.init(this)** 是VueRouter类中定义的init函数：
```
export default class VueRouter {
  // ...
  apps: Array<any>;
  // ...
  init (app: any) {
    process.env.NODE_ENV !== 'production' && assert(
      install.installed,
      `not installed. Make sure to call \`Vue.use(VueRouter)\` ` +
      `before creating root instance.`
    )
    // this.apps 保存持有 $options.router 属性的 Vue 实例，实际我们使用的时候只会new一个次VueRouter，所以通常this.apps的长度是1，但是vue支持多个
    this.apps.push(app)

    if (this.app) {
      return
    }
    
    // 这个app是根Vue实例
    this.app = app
    
    // 在new VueRouter的时候根据不同模式（history,hash,abstract）实例化的history实例
    const history = this.history

    if (history instanceof HTML5History) { // history
      history.transitionTo(history.getCurrentLocation())
    } else if (history instanceof HashHistory) { // hash
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
  // ...
}
```
history.transitionTo在history的基类中，代码在 **src/history/base.js**
```
transitionTo (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  const route = this.router.match(location, this.current)
  // ...
}
```
这里调用了this.router.match方法拿到当前路由，this.router.match返回了matcher的match函数：
```
export default class VueRouter {
  // ...
  match (
    raw: RawLocation,
    current?: Route,
    redirectedFrom?: Location
  ): Route {
    return this.matcher.match(raw, current, redirectedFrom)
  }
  // ...
}
```
那么matcher是怎么来的呢，在new VueRouter实例的时候通过 **createMatcher(options.routes || [], this)** 创建而来，这个函数返回了match函数和addRoutes函数。
### matcher
```
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
  ): Route {
    const location = normalizeLocation(raw, currentRoute, false, router)
    const { name } = location

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
        if (matchRoute(record.regex, location.path, location.params)) {
          return _createRoute(record, location, redirectedFrom)
        }
      }
    }
    // no match
    return _createRoute(null, location)
  }

  function redirect (
    record: RouteRecord,
    location: Location
  ): Route {
    const originalRedirect = record.redirect
    let redirect = typeof originalRedirect === 'function'
      ? originalRedirect(createRoute(record, location, null, router))
      : originalRedirect

    if (typeof redirect === 'string') {
      redirect = { path: redirect }
    }

    if (!redirect || typeof redirect !== 'object') {
      if (process.env.NODE_ENV !== 'production') {
        warn(
          false, `invalid redirect option: ${JSON.stringify(redirect)}`
        )
      }
      return _createRoute(null, location)
    }

    const re: Object = redirect
    const { name, path } = re
    let { query, hash, params } = location
    query = re.hasOwnProperty('query') ? re.query : query
    hash = re.hasOwnProperty('hash') ? re.hash : hash
    params = re.hasOwnProperty('params') ? re.params : params

    if (name) {
      // resolved named direct
      const targetRecord = nameMap[name]
      if (process.env.NODE_ENV !== 'production') {
        assert(targetRecord, `redirect failed: named route "${name}" not found.`)
      }
      return match({
        _normalized: true,
        name,
        query,
        hash,
        params
      }, undefined, location)
    } else if (path) {
      // 1. resolve relative redirect
      const rawPath = resolveRecordPath(path, record)
      // 2. resolve params
      const resolvedPath = fillParams(rawPath, params, `redirect route with path "${rawPath}"`)
      // 3. rematch with existing query and hash
      return match({
        _normalized: true,
        path: resolvedPath,
        query,
        hash
      }, undefined, location)
    } else {
      if (process.env.NODE_ENV !== 'production') {
        warn(false, `invalid redirect option: ${JSON.stringify(redirect)}`)
      }
      return _createRoute(null, location)
    }
  }

  function alias (
    record: RouteRecord,
    location: Location,
    matchAs: string
  ): Route {
    const aliasedPath = fillParams(matchAs, location.params, `aliased route with path "${matchAs}"`)
    const aliasedMatch = match({
      _normalized: true,
      path: aliasedPath
    })
    if (aliasedMatch) {
      const matched = aliasedMatch.matched
      const aliasedRecord = matched[matched.length - 1]
      location.params = aliasedMatch.params
      return _createRoute(aliasedRecord, location)
    }
    return _createRoute(null, location)
  }

  function _createRoute (
    record: ?RouteRecord,
    location: Location,
    redirectedFrom?: Location
  ): Route {
    if (record && record.redirect) {
      return redirect(record, redirectedFrom || location)
    }
    if (record && record.matchAs) {
      return alias(record, location, record.matchAs)
    }
    return createRoute(record, location, redirectedFrom, router)
  }

  return {
    match,
    addRoutes
  }
}
```
createMatcher主要就是返回了match和addRouters函数，match用来生成最新的路由记录record，addRoutes用来动态添加一条路由。  
在createMatcher中首先调用了createRouteMap函数：
```
export function createMatcher (
  routes: Array<RouteConfig>,
  router: VueRouter
): Matcher {
  // 传入routes列表
  // pathList: 是为了记录路由配置中的所有 path
  // pathMap: 通过 path 能快速查到对应的 RouteRecord
  // pathName: 通过 name 能快速查到对应的 RouteRecord
  const { pathList, pathMap, nameMap } = createRouteMap(routes)
  
  // 动态添加路由
  function addRoutes (routes) {
    createRouteMap(routes, pathList, pathMap, nameMap) // pathList, pathMap, nameMap此时的这三个参数是createMatcher函数开始通过createRouteMap方法获取赋值的
  }
  
  function match() {}
  // ...
  return {
    match,
    addRoutes
  }
}
```
看下createRouteMap函数：
```
// oldPathList,oldPathMap,oldNameMap是动态添加路由的时候会传入的参数，非动态添加的不传如这三个参数
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
  const pathList: Array<string> = oldPathList || []
  const pathMap: Dictionary<RouteRecord> = oldPathMap || Object.create(null)
  const nameMap: Dictionary<RouteRecord> = oldNameMap || Object.create(null)

  routes.forEach(route => {
    // 每条路由记录调用addRouteRecord添加路由
    addRouteRecord(pathList, pathMap, nameMap, route)
  })

  // ...

  return {
    pathList,
    pathMap,
    nameMap
  }
}

```
我们知道了createRouteMap中每条路由条目都是通过addRouteRecord来添加路由，下面看下addRouteRecord方法：
```
// 通过parent和child建立父子关系，注意通过addRoutes动态添加的只能是根级别的路由，不能在某个路由下面添加子路由，如果要添加子路由，则把父级及祖先级别都带上吧
function addRouteRecord (
  pathList: Array<string>,
  pathMap: Dictionary<RouteRecord>,
  nameMap: Dictionary<RouteRecord>,
  route: RouteConfig,
  parent?: RouteRecord,
  matchAs?: string
) {
  const { path, name } = route
  if (process.env.NODE_ENV !== 'production') {
    // ...
  }

  const pathToRegexpOptions: PathToRegexpOptions =
    route.pathToRegexpOptions || {}
  const normalizedPath = normalizePath(path, parent, pathToRegexpOptions.strict)

  if (typeof route.caseSensitive === 'boolean') {
    pathToRegexpOptions.sensitive = route.caseSensitive
  }
  
  // record记录
  const record: RouteRecord = {
    path: normalizedPath,
    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
    components: route.components || { default: route.component },
    instances: {},
    enteredCbs: {},
    name,
    parent,
    matchAs,
    redirect: route.redirect,
    beforeEnter: route.beforeEnter,
    meta: route.meta || {},
    props:
      route.props == null
        ? {}
        : route.components
          ? route.props
          : { default: route.props }
  }

  if (route.children) {
    // Warn if route is named, does not redirect and has a default child route.
    // If users navigate to this route by name, the default child will
    // not be rendered (GH Issue #629)
    if (process.env.NODE_ENV !== 'production') {
      // ...
    }
    
    // 如果有children，递归掉好用addRouteRecord方法添加路由
    route.children.forEach(child => {
      const childMatchAs = matchAs
        ? cleanPath(`${matchAs}/${child.path}`)
        : undefined
      addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs)
    })
  }
  // 最后添加到pathList和pathMap中
  if (!pathMap[record.path]) {
    pathList.push(record.path)
    pathMap[record.path] = record
  }
  // ...
}
```
#### 先梳理下流程
> 1. 开始通过new VueRouter创建了router实例，Vue.use(router)时调用install方法混入了router的breoreCreate和destroyed方法，breforeCreare方法中调用了router.init方法 在创建router实例时，首先通过createMatcher拿到matcher对象，matcher对象包含了match函数和addRoutes函数
> 2. 在createMatcher中调用createRouteMap方法建立路由父子关系，且得到了pathList, pathMap, nameMap三个对象
> 3. addRoutes方法也是通过createRouteMap添加路由，addRoutes只能添加根级别的路由
> 4. 刷新页面时，init方法最后调用了history.transitionTo实现路由跳转
> 5. 如果是点击router-link链接是通过history实例的push方法，push方法又调用了history.transitionTo实现路由的跳转
> 6. router-view只是一个占位的标签，用来展示匹配到的路由的内容
> 7. history.transitionTo方法是history基类中实现的方法，定义在 **src/history/base.js中**

看下transitionTo方法的定义
```
transitionTo (
    location: RawLocation,
    onComplete?: Function,
    onAbort?: Function
  ) {
    let route
    // catch redirect option https://github.com/vuejs/vue-router/issues/3201
    try {
    
      // 通过this.router.match方法拿到当前要跳转到的路由，实际是调用了this.router.matcher.match方法，这个方法由createMatcher返回（createMatcher还返回了addRoutes方法）
      // route 包含的属性如下：
      // name: location.name || (record && record.name),
      // meta: (record && record.meta) || {},
      // path: location.path || '/',
      // hash: location.hash || '',
      // query,
      // params: location.params || {},
      // fullPath: getFullPath(location, stringifyQuery),
      
      route = this.router.match(location, this.current)
    } catch (e) {
      this.errorCbs.forEach(cb => {
        cb(e)
      })
      // Exception should still be thrown
      throw e
    }
    const prev = this.current
    // 真正切换路由的地方
    this.confirmTransition(
      route,
      () => {
        this.updateRoute(route)
        onComplete && onComplete(route)
        this.ensureURL()
        this.router.afterHooks.forEach(hook => {
          hook && hook(route, prev)
        })

        // fire ready cbs once
        if (!this.ready) {
          this.ready = true
          this.readyCbs.forEach(cb => {
            cb(route)
          })
        }
      },
      err => {
        // ...
      }
    )
  }
```
看下match方法：
```
function match (
    raw: RawLocation,
    currentRoute?: Route,
    redirectedFrom?: Location
  ): Route {
    const location = normalizeLocation(raw, currentRoute, false, router)
    const { name } = location

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
        if (matchRoute(record.regex, location.path, location.params)) {
          return _createRoute(record, location, redirectedFrom)
        }
      }
    }
    // no match
    return _createRoute(null, location)
  }
```
match方法最后调用了_createRoute函数
```
function _createRoute (
    record: ?RouteRecord,
    location: Location,
    redirectedFrom?: Location
  ): Route {
  if (record && record.redirect) {
    return redirect(record, redirectedFrom || location)
  }
  if (record && record.matchAs) {
    return alias(record, location, record.matchAs)
  }
  return createRoute(record, location, redirectedFrom, router)
}
```
_createRoute函数又调用了createRoute函数：
```
export function createRoute (
  record: ?RouteRecord,
  location: Location,
  redirectedFrom?: ?Location,
  router?: VueRouter
): Route {
  const stringifyQuery = router && router.options.stringifyQuery

  let query: any = location.query || {}
  try {
    query = clone(query)
  } catch (e) {}
  
  // ***************************** router.matcher.match最终返回的route对象 *****************************
  const route: Route = {
    name: location.name || (record && record.name),
    meta: (record && record.meta) || {},
    path: location.path || '/',
    hash: location.hash || '',
    query,
    params: location.params || {},
    fullPath: getFullPath(location, stringifyQuery),
    matched: record ? formatMatch(record) : []
  }
  if (redirectedFrom) {
    route.redirectedFrom = getFullPath(redirectedFrom, stringifyQuery)
  }
  return Object.freeze(route)
}
```
回到tansitionTo方法里面，最后通过调用this.confirmTransition完成路径切换，由于这个过程可能有一些异步的操作（如异步组件），所以整个 confirmTransition API 设计成带有成功回调函数和失败回调函数。
```
confirmTransition (route: Route, onComplete: Function, onAbort?: Function) {
  const current = this.current
  this.pending = route
  // 终止函数
  const abort = err => {
    // changed after adding errors with
    // https://github.com/vuejs/vue-router/pull/3047 before that change,
    // redirect and aborted navigation would produce an err == null
    if (!isNavigationFailure(err) && isError(err)) {
      if (this.errorCbs.length) {
        this.errorCbs.forEach(cb => {
          cb(err)
        })
      } else {
        warn(false, 'uncaught error during route navigation:')
        console.error(err)
      }
    }
    // 调用终止回调
    onAbort && onAbort(err)
  }
  const lastRouteIndex = route.matched.length - 1
  const lastCurrentIndex = current.matched.length - 1
  // 如果route 和 current 是相同路径的话，直接调用 this.ensureUrl 和 abort
  if (
    isSameRoute(route, current) &&
    // in the case the route map has been dynamically appended to
    lastRouteIndex === lastCurrentIndex &&
    route.matched[lastRouteIndex] === current.matched[lastCurrentIndex]
  ) {
    this.ensureURL()
    return abort(createNavigationDuplicatedError(current, route))
  }

  // 通过resolveQueue拿到updated, deactivated, activated：其中this.current.matched和route.matched是当前路由和要跳转到的路由
  // updated：当前路由和要跳转的路由从0开始相同的部分
  // deactivated: 当前路由从不同索引（当前路由和要跳转到的路由从头开始第一个不同的索引）到结束
  // activate：要跳转到的路由从不同索引（当前路由和要跳转到的路由从头开始第一个不同的索引）到结束
  const { updated, deactivated, activate } = resolveQueue(
    this.current.matched,
    route.matched
  )

  // 执行一系列的钩子函数
  const queue: Array<?NavigationGuard> = [].concat(
    // 失活的组件调用离开守卫
    extractLeaveGuards(deactivated),
    // 调用全局的 beforeEach 守卫
    this.router.beforeHooks,
    // 重用的组件里调用 beforeRouteUpdate 守卫
    extractUpdateHooks(updated),
    // 激活的路由配置里调用 beforeEnter
    activated.map(m => m.beforeEnter),
    // 解析异步路由组件
    resolveAsyncComponents(activated)
  )
  // 迭代器，用来执行钩子的
  const iterator = (hook: NavigationGuard, next) => {
    if (this.pending !== route) {
      return abort(createNavigationCancelledError(current, route))
    }
    try {
      // hook就是queue里面的钩子
      hook(route, current, (to: any) => {
        if (to === false) {
          // next(false) -> abort navigation, ensure current URL
          this.ensureURL(true)
          abort(createNavigationAbortedError(current, route))
        } else if (isError(to)) {
          this.ensureURL(true)
          abort(to)
        } else if (
          typeof to === 'string' ||
          (typeof to === 'object' &&
            (typeof to.path === 'string' || typeof to.name === 'string'))
        ) {
          // next('/') or next({ path: '/' }) -> redirect
          abort(createNavigationRedirectedError(current, route))
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
  // 执行钩子队列
  runQueue(queue, iterator, () => {
    // wait until async components are resolved before
    // extracting in-component enter guards
    const enterGuards = extractEnterGuards(activated)
    const queue = enterGuards.concat(this.router.resolveHooks)
    runQueue(queue, iterator, () => {
      if (this.pending !== route) {
        return abort(createNavigationCancelledError(current, route))
      }
      this.pending = null
      onComplete(route)
      if (this.router.app) {
        this.router.app.$nextTick(() => {
          handleRouteEntered(route)
        })
      }
    })
  })
}
```
> confirmTransition 整个过程就是先判断路由是否切换，如果没有切换直接执行abort函数，如果切换了，则调用resolveQueue得到失活，重用，新激活的路由，根据这些路由组件了一个queue数组，数组有序的包含了 **①失活组件的离开钩子，②全局的 beforeEach 守卫，③重用的组件里调用 beforeRouteUpdate 守卫，④激活的路由配置里调用 beforeEnter 钩子，⑤解析异步路由组件** 这几项，然后执行runQueue把queue当做参数传入，runQueue借助一个step依次执行了queue的钩子，其中异步组件钩子的回调函数最后执行，借助了promise的then特性，最后调用了ensureUrl方法确保跳转的是当前的路由链接。











