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
#### Observer
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
#### 四.Watcher
```
export default class Watcher {
    // ...

    constructor(
        vm: Component,
        expOrFn: string | Function,
        cb: Function,
        options ? : ? Object,
        isRenderWatcher ? : boolean
    ) {
        this.vm = vm
        if (isRenderWatcher) {
            vm._watcher = this
        }
        vm._watchers.push(this)
        
        // ...
        
        // 新旧deps定义
        this.deps = []
        this.newDeps = []
        this.depIds = new Set()
        this.newDepIds = new Set()
        
        // ...
        
        if (typeof expOrFn === 'function') {
            // updateComponent赋值到this.getter
            this.getter = expOrFn
        } else {
        
            // ...
            
        }
        this.value = this.lazy ?
            undefined :
            this.get()
    }

    /**
     * Evaluate the getter, and re-collect dependencies.
     */
    get() {
        // 把当前的渲染watcher赋值给Dep.target(全局唯一一个)，并push进targetStack数组中
        pushTarget(this)
        let value
        const vm = this.vm
        try {
            // 执行 updateComponent 方法
            value = this.getter.call(vm, vm)
        } catch (e) {
            if (this.user) {
                handleError(e, vm, `getter for watcher "${this.expression}"`)
            } else {
                throw e
            }
        } finally {
            // "touch" every property so they are all tracked as
            // dependencies for deep watching
            if (this.deep) {
                traverse(value)
            }
            // 从targetStack数组中删除当前watcher，归还给它的父级watcher
            popTarget()
            this.cleanupDeps()
        }
        return value
    }

    /**
     * Add a dependency to this directive.
     */
    addDep(dep: Dep) {
        const id = dep.id
        if (!this.newDepIds.has(id)) {
            this.newDepIds.add(id)
            this.newDeps.push(dep)
            if (!this.depIds.has(id)) {
                // 想dep的subs增加该渲染watcher
                dep.addSub(this)
            }
        }
    }

    /**
     * Clean up for dependency collection.
     */
    // 清除没用到的dep
    cleanupDeps() {
        let i = this.deps.length
        while (i--) {
            const dep = this.deps[i]
            if (!this.newDepIds.has(dep.id)) {
                // 清除没用到的dep，为的是在一个视图隐藏的时候不触发更新（这时候没必要更新，因为视图已经隐藏了），每个属性都对应一个dep，每个dep.subs里面都管理这这个属性对应的watcher，如果属性                 // 所在的视图被隐藏了，就清除这个dep.subs里面的watcher，等到执行dep.notify的时候this.subs就为空，就不会继续执行派发更新了。
                dep.removeSub(this)
            }
        }
        let tmp = this.depIds
        this.depIds = this.newDepIds
        this.newDepIds = tmp
        this.newDepIds.clear()
        tmp = this.deps
        this.deps = this.newDeps
        this.newDeps = tmp
        this.newDeps.length = 0
    }

    /**
     * Subscriber interface.
     * Will be called when a dependency changes.
     */
    update() {
        /* istanbul ignore else */
        if (this.lazy) {
            this.dirty = true
        } else if (this.sync) {
            this.run()
        } else {
            // watcher队列，排列watcher
            queueWatcher(this)
        }
    }

    /**
     * Scheduler job interface.
     * Will be called by the scheduler.
     */
    // 在 src/core/observer/scheduler.js中定义的flushSchedulerQueue中调用该方法，或者在上面的update方法中直接调用this.run()调用该方法，完成dom更新。
    run() {
        if (this.active) {
            // 派发更新的时候调用watcher.run执行watcher的get方法，从而重新渲染
            const value = this.get()
            
            // ...
            
        }
    }
    
    // ...
    
}
```
#### 五.queueWatcher方法
```
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      // 把当前watcher push进queue数组中
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      // nextTick异步执行flushSchedulerQueue方法
      nextTick(flushSchedulerQueue)
    }
  }
}
```
#### 六.flushSchedulerQueue
```
function flushSchedulerQueue () {
  currentFlushTimestamp = getNow()
  flushing = true
  let watcher, id

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  // 重排watcher
  queue.sort((a, b) => a.id - b.id)

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    // 如果有before则执行回调，渲染watcher的这个函数是执行了beforeUpdate钩子，
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run() // 这里调用watcher.run从而完成dom的更新
    // in dev build, check and stop circular updates.
    
    // ...
    
  }

  // ...
  
}
```
#### 七.nextTick
数据的更新并不会立即映射到dom上，例如：
```
<template>
  <div id="test">{{ msg }}</div>
</template>
<script>
export default {
  data() {
    return {
      msg: 'hello'
    }
  },
  mounted() {
    this.msg = 'vue';
    console.lo(document.getElementById('test').innerText) // 这时候打印的仍是hello
  }
}
</script>
```
这是因为vue源码在派发更新queueWatcher的时候使用了nextTick方法，这个方法定义在‘src/core/util/next-tick.js’中：  
```
/* @flow */
/* globals MutationObserver */

import { noop } from 'shared/util'
import { handleError } from './error'
import { isIE, isIOS, isNative } from './env'

export let isUsingMicroTask = false

const callbacks = []
let pending = false

function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]() // 调用 flushSchedulerQueue
  }
}

let timerFunc

// 定义timerFunc异步执行的函数，执行flushCallbacks， flushSchedulerQueue 在 flushCallbacks 中被调用
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  // Use MutationObserver where native Promise is not available,
  // e.g. PhantomJS, iOS7, Android 4.4
  // (#6466 MutationObserver is unreliable in IE11)
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  // Fallback to setImmediate.
  // Technically it leverages the (macro) task queue,
  // but it is still a better choice than setTimeout.
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}

export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  // 先放入callbacks数组中
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    // 执行函数，timerFunc中定义了异步任务，调用callbacks数组中的 flushSchedulerQueue 函数更新
    timerFunc()
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}

```
由于js单线程机制，js的运行遵循着js事件循环机制：  
> 1.所有同步任务都在主线程上执行，形成一个执行栈（execution context stack）。  
2.主线程之外，还存在一个"任务队列"（task queue）。只要异步任务有了运行结果，就在"任务队列"之中放置一个事件。  
3.一旦"执行栈"中的所有同步任务执行完毕，系统就会读取"任务队列"，看看里面有哪些事件。那些对应的异步任务，于是结束等待状态，进入执行栈，开始执行。  
4.主线程不断重复上面的第三步。

**nextTick** 就是利用了异步任务来执行，并且 **queueWatcher** 执行异步任务nextTick是waiting为false的时候才执行，且执行前把waiting赋值为true，这样在dom渲染之前watcher的flushSchedulerQueue只执行一次，避免了 **一次事件循环中** 多次修改数据多次渲染，浪费性能。

#### 再次总结：
> 收集依赖的目的是为了当这些响应式数据发生变化，触发它们的 **setter** 的时候，能知道应该通知哪些订阅者(watcher)去做相应的逻辑处理，我们把这个过程叫派发更新

> 派发更新的过程实际上就是当数据发生变化的时候，触发 **setter** 逻辑，把在依赖过程中订阅的所有观察者，也就是 **watcher** ，都触发它们的 **update** 过程，这个过程又利用了队列做了进一步优化，在 **nextTick** 后执行所有 **watcher** 的 **run**，最后执行它们的回调函数。

> **nextTick** 利用异步任务和waiting字段，**一次事件循环中** 多次修改只做一次渲染，提高了性能。
