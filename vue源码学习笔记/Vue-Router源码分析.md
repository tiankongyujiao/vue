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
        // 同过new Vue传入的router实例（new VueRouter创建）
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
在install中执行注入的beforeCreate钩子中执行的 **this._router.init(this)** 是VueRouter中定义的init函数：
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
##### history.transitionTo在history的基类中，代码在 **src/history/base.js**
```
transitionTo (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  const route = this.router.match(location, this.current)
  // ...
}
```
