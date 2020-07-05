### vue源码学习笔记--合并配置和生命周期
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
其中if进去是子组件的逻辑，通过if的 options._isComponent 可以看出是个组件实例，这时候调用initInternalComponent(vm, options)合并配置，这个合并配置相对简单，因为子组件的合并在通过Vue.extend创建子组件实例的时候已经通过mergeOptions合并了一层（合并了大Vue以及子组件对象options），这次合并即这里我说的第三个地方的合并。在回到上面的代码，如果不是子组件实例，则会走到else逻辑，调用mergeOptions方法合并配置。从上面我们可以看到合并配置使用的主要就是mergeOptions方法，下面分析这个方法：
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
mergeHook的实现是使用了两层三元运算符，如果不存在childVal，直接返回parentVal，如果存在childVal切存在parentVal，则吧childVal合并到parentVal后返回，如果存在childVal，不存在parentVal，则把childVal放入一个数组返回一个只有一个元素的数组，总之mergeOptions最终返回的都是一个数组，直接返回的parentVal其实也是一个之前就合并过的数组。   

所有的属性合并完成以后最终到放到options对象中返回。




