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
