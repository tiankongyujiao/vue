### vue源码学习笔记--深入响应式原理
#### 先总结：
+ vue响应式原理是利用Object.defineProperty(obj, key, descriptor)的get和set实现的，在get中实现收集依赖，set中实现派发更新。  
+ vue的响应式原理在源码中依赖于Watcher，Dep，Observer类，其中Watcher又分为渲染Watcher，user Watcher，内部Watcher，这里我们主要看渲染Watcher。  
+ Watcher是订阅者，存储依赖，等set的时候再通过notify触发watcher的get，调用update更新dom。  
+ Dep是对Watcher的管理，Dep的实例dep，dep.subs数组中存放的是Watchers订阅者，dep.id是每个属性的Dep的id，Dep脱离Watcher单独存在没有意义。  
+ Observer类定义Object.defineProperty的地方。  
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



