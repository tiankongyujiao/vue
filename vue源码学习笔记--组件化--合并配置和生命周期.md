### vue源码学习笔记--组件化--合并配置和生命周期
#### 一.合并配置
合并配置我理解的主要发生在三个地方，其中两个是在_init()方法里面：
```
Vue.prototype._init = function (options?: Object) {
  // ...
  if (options && options._isComponent) {
    // optimize internal component instantiation
    // since dynamic options merging is pretty slow, and none of the
    // internal component options needs special treatment.
    initInternalComponent(vm, options)
  } else {
    vm.$options = mergeOptions(
      resolveConstructorOptions(vm.constructor),
      options || {},
      vm
    )
  }
  // ...
}
```
其中if进去是子组件的逻辑，通过if的 options._isComponent 可以看出是个组件实例，这时候调用initInternalComponent(vm, options)合并配置，这个合并配置相对简单，因为子组件的合并在通过Vue.extend创建子组件实例的时候已经通过mergeOptions合并了一层（合并了大Vue以及子组件对象options），这次合并即这里我说的第三个地方的合并。在回到上面的代码，如果不是子组件实例，则会走到else逻辑，调用mergeOptions方法合并配置。  
先来看else的逻辑，在执行new Vue的时候回走到else的逻辑，resolveConstructorOptions(vm.constructor)的返回值在这里就是Vue.options，这个值在initGlobalAPI(Vue)中定义，这个函数在‘src/core/global-api/index.js’中定义：
```
export function initGlobalAPI (Vue: GlobalAPI) {
  // ...
  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })

  // this is used to identify the "base" constructor to extend all plain-object
  // components with in Weex's multi-instance scenarios.
  Vue.options._base = Vue

  extend(Vue.options.components, builtInComponents)
  // ...
}
```
开始创建了一个空对象赋值给Vue.options，然后通过ASSET_TYPES.forEach这句把components,directives,filters加到Vue.options上面，然后把_base加到Vue.options上，这个_base我们在实例化子组件的时候用到的。然后把一些内置的组建，Vue中内置的组建有keep-alive, transition, transition-group组建挂到Vue.options.components上面，这也是我们不注册这三个组建就能使用的原因。

从上面我们可以看到合并配置使用的主要就是*mergeOptions*方法，下面分析这个方法：
```
/**
 * Merge two option objects into a new one.
 * Core utility used in both instantiation and inheritance.
 */
export function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  if (process.env.NODE_ENV !== 'production') {
    checkComponents(child)
  }

  if (typeof child === 'function') {
    child = child.options
  }

  normalizeProps(child, vm)
  normalizeInject(child, vm)
  normalizeDirectives(child)

  // Apply extends and mixins on the child options,
  // but only if it is a raw options object that isn't
  // the result of another mergeOptions call.
  // Only merged options has the _base property.
  if (!child._base) {
    if (child.extends) {
      parent = mergeOptions(parent, child.extends, vm)
    }
    if (child.mixins) {
      for (let i = 0, l = child.mixins.length; i < l; i++) {
        parent = mergeOptions(parent, child.mixins[i], vm)
      }
    }
  }

  const options = {}
  let key
  for (key in parent) {
    mergeField(key)
  }
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  return options
}
```
+ 上面起合并配置作用的主要是下面半段，上面半段主要是extends 和 mixins 合并到 parent 上，然后遍历 parent，调用 mergeField，然后再遍历 child，如果 key 不在 parent 的自身属性上，则调用 mergeField。  
+ mergeField对于不同的key有着不同的合并策略，这些合并策略都存储在strats数组中，如果数组中不存在，则使用默认的合并策略defaultStrat。  
+ strats是在‘src/core/config.js’中定义，然后再‘src/core/utils/options.js’中扩展赋值，其中有el，data，生命周期hook，component，directive，filter，watch，props，methods，inject，computed，provide的合并策略，这里我们主要介绍生命周期钩子的合并策略，即mergeHook方法。
```
function mergeHook (
  parentVal: ?Array<Function>,
  childVal: ?Function | ?Array<Function>
): ?Array<Function> {
  const res = childVal
    ? parentVal
      ? parentVal.concat(childVal)
      : Array.isArray(childVal)
        ? childVal
        : [childVal]
    : parentVal
  return res
    ? dedupeHooks(res)
    : res
}
LIFECYCLE_HOOKS.forEach(hook => {
  strats[hook] = mergeHook
})
```
其中的LIFECYCLE_HOOKS定义了组件的所有生命周期钩子，定义在‘src/shared/constants.js’中，有下面这些钩子函数：beforeCreate,created,beforeMount,mounted,beforeUpdate,updated,beforeDestroy,activated,deactivated,errorCaptured.  
mergeHook的实现是使用了两层三元运算符，如果不存在childVal，直接返回parentVal，如果存在childVal且存在parentVal，则把childVal合并到parentVal后返回，如果存在childVal，不存在parentVal，则把childVal放入一个数组返回一个只有一个元素的数组，总之mergeOptions最终返回的都是一个数组，直接返回的parentVal其实也是一个之前就合并过的数组。   

所有的属性合并完成以后最终到放到options对象中返回。

分析完了else的逻辑，再来看看if是组件实例的合并配置：  
前面提到过子组件实例的合并配置在Vue.extend中就完成了主要配置（把大Vue和传入的配置options做合并），if逻辑里面又调用了initInternalComponent这个方法做又一次的合并，下面看下这个方法：
```
export function initInternalComponent (vm: Component, options: InternalComponentOptions) {
  const opts = vm.$options = Object.create(vm.constructor.options)
  // doing this because it's faster than dynamic enumeration.
  const parentVnode = options._parentVnode
  opts.parent = options.parent
  opts._parentVnode = parentVnode

  const vnodeComponentOptions = parentVnode.componentOptions
  opts.propsData = vnodeComponentOptions.propsData
  opts._parentListeners = vnodeComponentOptions.listeners
  opts._renderChildren = vnodeComponentOptions.children
  opts._componentTag = vnodeComponentOptions.tag

  if (options.render) {
    opts.render = options.render
    opts.staticRenderFns = options.staticRenderFns
  }
}
```
方法内的第一行的vm.constructor就是指子组件Sub的构造函数，相当于*vm.$options = Object.create(Sub.options)*，接着又把实例化子组件传入的子组件父 VNode 实例 parentVnode、子组件的父 Vue 实例 parent 保存到 vm.$options 中，另外还保留了 parentVnode 配置中的如 propsData 等其它的属性，这么看来，initInternalComponent 只是做了简单一层对象赋值，并不涉及到递归、合并策略等复杂逻辑。

#### 二.生命周期
每个Vue实例在被创建的时候都要经过一系列的初始化工作：设置数据监听，编译模板，挂在实例到DOM，数据变化时更新DOM。在这个过程中会执行一些列的生命周期钩子，让我们能够在特定的场景下添加自己的代码。  
源码中最终执行生命周期钩子是通过调用callHook方法：
```
export function callHook (vm: Component, hook: string) {
  // #7573 disable dep collection when invoking lifecycle hooks
  pushTarget()
  const handlers = vm.$options[hook]
  const info = `${hook} hook`
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      invokeWithErrorHandling(handlers[i], vm, null, vm, info)
    }
  }
  if (vm._hasHookEvent) {
    vm.$emit('hook:' + hook)
  }
  popTarget() 
}
```
callHook 函数就是：根据传入的字符串 hook，去拿到 vm.$options[hook] 对应的回调函数数组（上面我们提到过合并后的回调钩子都是一个数组），然后遍历执行，执行的时候把 vm 作为函数执行的上下文。  
每个生命周期钩子调用的时机：  
+ beforeCreate: 实例化 Vue 的阶段，初始化生命周期，事件，渲染函数之后，初始化 props、data、methods、watch、computed 等属性之前。获取不到data。不能够访问 DOM，可以和后台交互数据。执行顺序先父后子。
+ created:  实例化 Vue 的阶段，初始化 props、data、methods、watch、computed 等属性之后。能获取到data。不能够访问 DOM，可以和后台交互数据。执行顺序先父后子。
+ beforeMount:  DOM 挂载之前，它的调用时机是在 mountComponent 函数中，定义在 src/core/instance/lifecycle.js 中。不可以操作DOM。执行顺序先父后子。
+ mounted: DOM 挂载之后，执行顺序先子后父。可以操作DOM。
+ beforeUpdate: beforeUpdate 的执行时机是在渲染 Watcher 的 before 函数中，发生在数据更新前。
+ updated: 发生在数据更新后。
+ beforeDestory: 组件销毁的阶段前。
+ destoryed: 组件销毁的阶段后。
+ activated 和 deactivated 钩子函数是专门为 keep-alive 组件定制的钩子

