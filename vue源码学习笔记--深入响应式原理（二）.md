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
