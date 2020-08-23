### vue源码学习笔记--深入响应式原理（set，computed，watch）分析
#### 一. Vue.set
vue对于没有在data中添加的属性，当去设置它的值时是不会触发setter的，也就是新添加的属性它并不是一个响应式对象，如果想让它成为响应式对象，就需要使用 **Vue.set**方法。  
```
Vue.set = set
```
这个方法的定义是在src/core/global-api/index.js中，set又是在src/core/observer/index.js中定义：
```
export function set (target: Array<any> | Object, key: any, val: any): any {
  // 参数tartet：要定义的目标对象，key：要定义的目标对象的属性或者数组的下标，val：要定义的目标对象的属性的值
  
  // 如果是开发环境，且target未定义或者target是基本类型：string，number，symbol，boolean，则给出warn提示
  if (process.env.NODE_ENV !== 'production' &&
    (isUndef(target) || isPrimitive(target))
  ) {
    warn(`Cannot set reactive property on undefined, null, or primitive value: ${(target: any)}`)
  }
  // 如果target是一个数组且下标合法，直接通过splice 去添加进数组然后返回（splice是数组的变异方法，vue源码对他有进行了一层封装，后面介绍）
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    target.splice(key, 1, val)
    return val
  }
  // 如果key在target中了，且key不是原型上的属性，说明已经存在target上面，已经是响应式了，则直接赋值并返回val，
  if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return val
  }
  // 获取target.__ob__，它是在 Observer 的构造函数执行的时候初始化的，表示 Observer 的一个实例，如果它不存在，则说明 target 不是一个响应式的对象，则直接赋值并返回
  const ob = (target: any).__ob__
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid adding reactive properties to a Vue instance or its root $data ' +
      'at runtime - declare it upfront in the data option.'
    )
    return val
  }
  if (!ob) {
    target[key] = val
    return val
  }
  // 把新添加的属性变成响应式对象，然后再通过 ob.dep.notify() 手动的触发依赖通知
  defineReactive(ob.value, key, val)
  ob.dep.notify()
  return val
}

```
最后能够通过ob.dep.notify()触发更新，是因为在Object.defineProperty的get中有这么一段收集子属性的依赖（只有先收集了依赖，才能触发更新）：
```
Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) { // 这里收集子属性的依赖
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      // ...
    }
  })
```
##### 数组的特殊性
1. 直接利用索引修改数组无效，例如：**vm.items[indexOfItem] = newValue**，可以改为使用：**Vue.set(example1.items, indexOfItem, newValue)**
2. 直接修改数组长度无效，例如：**vm.items.length = newLength**，可以改为使用：**target.splice(key, 1, val)** 。
上面的splice为啥会使数组变为响应式呢，这就是我们上面提到的，vue源码对数组的 **'push','pop', 'shift','unshift','splice','sort','reverse'** 做了重写，变异了数组的方法，在变异的方法内部调用了 **ob.dep.notify()** 方法，从而触发dom更新的，看下具体代码：
```
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      if (hasProto) {
        // 一般都会走到这个逻辑
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }
  // ...
}

```
然后看下protoAugment方法：
```
function protoAugment (target, src: Object) {
  /* eslint-disable no-proto */
  target.__proto__ = src
  /* eslint-enable no-proto */
}
```
这里就是把arr赋值给数组的原型，这个arr就是调用 **protoAugment** 传入的arrMethods，这个方法和对数组的重写都是定义在src/core/observer/array.js中：
```
import { def } from '../util/index'

const arrayProto = Array.prototype
export const arrayMethods = Object.create(arrayProto) // arrayMethods 首先继承了 Array

const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]

/**
 * Intercept mutating methods and emit events
 */
// 然后对数组中所有能改变数组自身的方法，如 push、pop 等这些方法进行重写
methodsToPatch.forEach(function (method) {
  // cache original method
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator (...args) {
    const result = original.apply(this, args)
    const ob = this.__ob__
    console.log(ob, '__ob__')
    let inserted
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    if (inserted) ob.observeArray(inserted)
    // notify change
    ob.dep.notify()
    return result
  })
})

```
> 重写后的方法会先执行它们本身原有的逻辑，并对能增加数组长度的 3 个方法 **push、unshift、splice** 方法做了判断，获取到插入的值，然后把新添加的值变成一个响应式对象，并且再调用 **ob.dep.notify()** 手动触发依赖通知，这就很好地解释了之前的示例中调用 **vm.items.splice(newLength)** 方法可以检测到变化。

#### 二.computed
vue计算属性computed的源码分析：  
vue计算属性computed的初始化也是在instate（instate在vue的_init方法中调用的，vue的_init方法又是在Vue中调用的，当new一个vue实例时调用）中，initState有这么一句：
```
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed) // 这一句
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```
然后是initComputed方法：
```
function initComputed (vm: Component, computed: Object) {
  // $flow-disable-line
  // 给当前的vm实例定义了_computedWatchers空对象，使用Object.create(null)创建的对象不具有对象本身具有的原型属性，_computedWatchers在computedGetter（计算属性的get）中使用
  const watchers = vm._computedWatchers = Object.create(null)
  // computed properties are just getters during SSR
  const isSSR = isServerRendering() // web端渲染为false
  
  // 遍历computed对象
  for (const key in computed) {
    // 拿到键对应的值（通常是一个函数）
    const userDef = computed[key]
    // 如果userDef不是一个函数，说明它是一个对象且有get方法，这里取的键对应的键get
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(
        `Getter is missing for computed property "${key}".`,
        vm
      )
    }
    // 如果不是服务端渲染，则 new 一个 Watcher，放入 vm._computedWatchers中，供后面该键属性的get（这个get是使用Object.defineProperty重新定义了一次，我们定义的get在这个重新定义的       // get里面调用）使用
    if (!isSSR) {
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }

    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here.
    // 如果是组件的话，这个逻辑是进不去的，因为在创建组件实例的时候，即在 Vue.extend 方法中就已经把该键存取到了vue的原型上，所以如果是组件vm，这里不会进入到if中
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    } else if (process.env.NODE_ENV !== 'production') {
      if (key in vm.$data) {
        warn(`The computed property "${key}" is already defined in data.`, vm)
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(`The computed property "${key}" is already defined as a prop.`, vm)
      }
    }
  }
}
```
注释中说到组件的话不会调用defineComputed方法，组件在创建vm的实例的时候就已经通过下面语句定义到了vue的原型里：
```
  Vue.extend = function (extendOptions: Object): Function {
    // ...
    if (Sub.options.computed) {
      initComputed(Sub)
    }
    // ...
  }
```
在调用 **Vue.extend** 创建组件vm的时候，如果组件有 **computed** 属性，就通过 **initComputed()** 方法定义了该计算属性，注意这里的 **initComputed** 不是 **src/core/instance/state.js** 中的 **initComputed**, 是 **src/core/global-api/extend.js** 中的:
```
function initComputed (Comp) {
  const computed = Comp.options.computed
  for (const key in computed) {
    defineComputed(Comp.prototype, key, computed[key]); // 注意这里传入的是当前vm的原型，就是把组件的computed的属性都定义到了vm的原型上面
  }
}

```
在这里遍历了computed对象里的所有属性，并调用了 **defineComputed**，这个方法定义在 **src/core/instance/state.js** 中：
```
export function defineComputed (
  target: any, // vm原型
  key: string, // computed 的键
  userDef: Object | Function // computed 的键值
) {
  const shouldCache = !isServerRendering()
  // 通常来说userDef都是一个函数，因为computed一般没有set，这里只看这种情况
  if (typeof userDef === 'function') {
    // 这里会调用 createComputedGetter 方法，因为不是服务端渲染，shouldCache为true
    sharedPropertyDefinition.get = shouldCache
      ? createComputedGetter(key)
      : createGetterInvoker(userDef)
    sharedPropertyDefinition.set = noop
  } else {
    sharedPropertyDefinition.get = userDef.get
      ? shouldCache && userDef.cache !== false
        ? createComputedGetter(key)
        : createGetterInvoker(userDef.get)
      : noop
    sharedPropertyDefinition.set = userDef.set || noop
  }
  if (process.env.NODE_ENV !== 'production' &&
      sharedPropertyDefinition.set === noop) {
    sharedPropertyDefinition.set = function () {
      warn(
        `Computed property "${key}" was assigned to but it has no setter.`,
        this
      )
    }
  }
  // 通过 Object.defineProperty把键和包装后的get定义到vm原型上
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```
下面看下 **createComputedGetter** 方法：
```
function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```
这个方法的定义返回了 **computedGetter** ，这个方法就是computed的最终get，当访问computed的属性的时候就会调用 **computedGetter** 方法。  
在 **computedGetter** 方法里面首先获得了当前vm实例的 **_computedWatchers**，这个上面我们说过就是在computed初始话的时候 **new Watcher** 塞进去的，然后调用 **watcher.evaluate** ：
```
evaluate () {
  this.value = this.get()
  this.dirty = false
}
```
+ 这个方法调用了 **this.get** ，在get中调用了 **value = this.getter.call(vm, vm)** ，**this.getter** 就是我们自己定义的计算属性的get方法，执行计算得到计算属性的值，执行计算的过程冲又会触发依赖属性的get。  
+ 到这里计算属性的get就分析完了，一般情况是没有set的，下面是set的情况：  
+ 上面 **computedGetter** 中后面又调用了 **watcher.depend** 方法，此时的 **Dep.target** 是组件的渲染 **watcher** ，而 **watcher** 是当前访问的计算属性的 **computed watcher** 。  **watcher.depend** 的作用是往当前渲染 **watcher** 的 **newDeps** 和 **newDepIds** 上添加当前计算属性所依赖的 **data** 属性的 **dep**，并且在当前计算属性所依赖的 **data** 属性的 **dep** 上添加渲染 **watcher** ，这样在 **set** 的时候估计就能找到当前的 **watcher** 完成更新（没有深入研究）。
