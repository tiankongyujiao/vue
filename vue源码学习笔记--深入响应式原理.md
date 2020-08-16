### vue源码学习笔记--深入响应式原理
+ vue响应式原理是利用Object.defineProperty(obj, key, descriptor)的get和set实现的，通过get中实现收集依赖，set中实现派发更新。  
+ vue的响应式原理在源码中依赖于Watcher，Dep，Observer类，其中Watcher又分为渲染Watcher，user Watcher，内部Watcher，这里我们主要看渲染Watcher。  
+ Watcher是订阅者，存储依赖，等set的时候再通过notify触发watcher的get，调用update更新dom。  
+ Dep是对Watcher的管理，Dep的实例dep，dep.subs数组中存放的是Watchers订阅者，dep.id是每个属性的Dep的id，Dep脱离Watcher单独存在没有意义。  
