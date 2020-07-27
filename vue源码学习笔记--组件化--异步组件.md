### vue源码学习笔记--组件化--异步组件
异步组件注册有三种方式：1.工厂函数 2.Promise 创建组件 3.高级异步组件  
#### 1.工厂函数方式
使用方式如下：
```
Vue.component('hello-world', function(resolve,reject){
  require(['./components/HelloWorld.vue'], function(res){
  	resolve(res)
  })
})
```
上面使用工厂函数的方式定义了helloWorld异步组件，在调用helloWorld的地方，创建虚拟dom时，调用了resolveAsset方法，解析出来的就是我们这里定义的工厂函数，然后执行createComponent方法，在createComponent中有这么一段逻辑：
```
// ...
  let asyncFactory
  if (isUndef(Ctor.cid)) {
    asyncFactory = Ctor
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor)
    if (Ctor === undefined) {
      // return a placeholder node for async component, which is rendered
      // as a comment node but preserves all the raw information for the node.
      // the information will be used for async server-rendering and hydration.
      return createAsyncPlaceholder(
        asyncFactory,
        data,
        context,
        children,
        tag
      )
    }
  }
// ..
```
这里Ctor其实就是我们定义的工厂函数，所以它的cid不存在，进入if逻辑，执行resolveAsyncComponent方法，加载异步组件，其中这行代码就是加载了异步组件：*const res = factory(resolve, reject)*，这时执行了require，由于js的单线程，会继续执行resolveAsyncComponent，此时返回的是一个空的注释节点。  
同步代码执行完了，会执行require的回调，resolve方法被执行，把resolve缓存到factory.resolve上面，然后执行forceRender方法，forceRender遍历owners执行$forceUpdate方法，调用渲染watch，重新渲染， 又走到了_createElement方法，后面再调用resolveAsyncComponent返回的就是异步组件的构造器factory.resolve，后面的逻辑就和同步是一样的了。
