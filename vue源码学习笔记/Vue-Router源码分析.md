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
      if (isDef(this.$options.router)) {
        this._routerRoot = this // this._routerRoot开始是Vue，对子组件而言，this._routerRoot始终指向的离它最近的传入了 router 对象作为配置而实例化的父实例
        this._router = this.$options.router // 同过new Vue传入的router实例（new VueRouter创建）
        this._router.init(this) // 初始化路由，带入Vue参数
        Vue.util.defineReactive(this, '_route', this._router.history.current) // 把 this._route 变成响应式对象
      } else {
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

  Vue.component('RouterView', View)
  Vue.component('RouterLink', Link)

  const strats = Vue.config.optionMergeStrategies
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
}
```
