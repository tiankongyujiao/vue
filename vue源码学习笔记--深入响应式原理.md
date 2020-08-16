### vue源码学习笔记--深入响应式原理
#### 先总结：
+ vue响应式原理是利用 **Object.defineProperty(obj, key, descriptor)** 的 **get** 和 **set** 实现的，在 **get** 中实现收集依赖，**set** 中实现派发更新。  
+ vue的响应式原理在源码中依赖于 **Watcher，Dep，Observer** 类，其中 **Watcher** 又分为渲染 **Watcher，user Watcher** ，内部 **Watcher** ，这里我们主要看渲染 **Watcher** 。  
+ **Watcher** 是订阅者，存储依赖，等 **set** 的时候再通过 **notify** 触发 **watcher** 的 **get** ，调用 **update** 更新 **dom**。  
+ **Dep** 是对 **Watcher** 的管理，**Dep** 的实例 **dep** ，**dep.subs** 数组中存放的是 **Watchers** 订阅者，**dep.id** 是每个属性的 **Dep** 的id，**Dep** 脱离 **Watcher** 单独存在没有意义。 
+ **Observer** 类定义 **Object.defineProperty** 的地方。  
#### 一.响应式对象
```
Object.defineProperty(obj, prop, descriptor)
// 这个方法会直接在一个对象上定义一个新属性，或者修改一个对象的现有属性， 并返回这个对象
// obj 是要在其上定义属性的对象；prop 是要定义或修改的属性的名称；descriptor 是将被定义或修改的属性描述符。
```
这里我们最关心的是 **get** 和 **set**，**get** 是一个给属性提供的 **getter** 方法，当我们访问了该属性的时候会触发 **getter** 方法；**set** 是一个给属性提供的 **setter** 方法，当我们对该属性做修改的时候会触发 **setter** 方法。
#### 二.在哪里调用的Object.defineProperty，实现双向数据绑定的
在Vue初试化的过程中会调用 **_init** 方法，**_init** 方法中会调用 **_initState** 方法：
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
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```
**initState** 方法主要是对 **props、methods、data、computed 和 wathcer** 等属性做了初始化操作。这里我们重点分析 **props** 和 **data**。  
**initProp** 和 **initData** 都通过 **proxy** 把每一个值 vm._props.xxx 都代理到 vm.xxx 上，把每一个值 vm._data.xxx 都代理到 vm.xxx 上。  
然后 **initProp** 和 **initData**方法中都最中通过调用 **defineReactive** 方法来定义响应式：
```
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  // 每个属性的dep实例，在闭包范围内的dep
  const dep = new Dep()

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }
  
  // 如果val也是一个对象，则继续调用observe方法，返回val的Observer对象childOb
  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        // 收集依赖调用方法
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      // 如果修改的结果newVal是一个对象，则调用observe再次对该对象做响应式处理，返回childOb这个Observer对象
      childOb = !shallow && observe(newVal)
      // 派发更新调用方法
      dep.notify()
    }
  })
}
```
#### 三.Dep
```
export default class Dep {
  static target: ?Watcher; // 静态属性 target，这是一个全局唯一 Watcher，这是一个非常巧妙的设计，因为在同一时间只能有一个全局的 Watcher 被计算
  id: number; // 每个dep实例的id自增
  subs: Array<Watcher>; // 自身属性 subs 也是 Watcher 的数组

  constructor () {
    this.id = uid++ // 初始化id
    this.subs = [] // 初始化subs
  }

  addSub (sub: Watcher) {
    this.subs.push(sub) // 通过watcher的dep.addSub(this)方法调用，在dep.subs中添加一个watcher对象
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }
  // 在Object.defineProperty的get中调用的方法，用来收集依赖
  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }
  // 在Object.defineProperty的set中调用的方法，用来派发更新
  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      // 调用watcher的update方法
      subs[i].update()
    }
  }
}
```
