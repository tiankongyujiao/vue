### reactive, ref
通过reactive和ref包装一层，把字符串包装进ref，或者把对象包装进reactive中，ref也可以包装对象，但是内部还是会用reactive包装一层的，调用响应式的reactive和ref方法。  
ref调用了createRef方法，直接在createRef方法内设置了get和set函数，在get内track收集依赖，在set方法内触发更新trigger。  
reactive是使用了new Proxy，同样也是在get内track收集依赖，在set方法内触发更新trigger

### track收集依赖

### trigger触发更新

### effect副作用函数，当依赖发生变化，触发副作用函数执行
effect副作用函数的实现原理：  
当执行effect(fn)的时候会第一次执行到fn，看一下它的实现：
```
const effect = function reactiveEffect(): unknown {
    if (!effect.active) {
      return options.scheduler ? undefined : fn()
    }
    if (!effectStack.includes(effect)) {
      cleanup(effect)
      try {
        enableTracking()
        effectStack.push(effect)
        activeEffect = effect // 执行fn之前把当前的activeEffect赋值为effect
        return fn()
      } finally {
        effectStack.pop()
        resetTracking()
        activeEffect = effectStack[effectStack.length - 1]
      }
    }
  } as ReactiveEffect
```
在执行fn之前把当前的activeEffect赋值为effect，这时候fn中的响应式的变量被访问了就会触发变量的get操作，看下get，get的执行会触发track的过程，看下track部分逻辑：
```
if (!dep.has(activeEffect)) {
  dep.add(activeEffect)
  activeEffect.deps.push(dep)
  //当一个 reactive 对象属性或一个 ref 作为依赖被追踪时，将调用 onTrack
  // onTrack 只有在开发环境才会生效
  if (__DEV__ && activeEffect.options.onTrack) {
    activeEffect.options.onTrack({
      effect: activeEffect,
      target,
      type,
      key
    })
  }
}
```
如果当前的响应式变量的dep中没有副作用函数的effect，则添加这个渲染函数的effect，渲染函数的effect的deps也添加了当前的这个响应式的变量。  
然后在响应式对象改变的时候会触发set，set会触发trigger的过程，看下trigger的部分逻辑：
```
const depsMap = targetMap.get(target);
// ...
// 计算得到所有的effect

const run = (effect: ReactiveEffect) => {
  if (__DEV__ && effect.options.onTrigger) {
    effect.options.onTrigger({
      effect,
      target,
      key,
      type,
      newValue,
      oldValue,
      oldTarget
    })
  }
  if (effect.options.scheduler) {
    effect.options.scheduler(effect)
  } else {
    effect()
  }
}
// 然后执行所有effect的run方法
effects.forEach(run)
```
这里执行effect方法，进而又执行了副作用函数；当然里面还有一系列的判断重复的过程，比如再次访问响应式变化，依赖收集的过程又执行一遍，这时已有的effect不再重新添加。  
响应式变量的dep是存储在一个全局变量targetMap中的，是一个map对象，值中又存储了depsMap对象。




