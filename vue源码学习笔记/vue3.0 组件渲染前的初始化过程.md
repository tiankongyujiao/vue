### 组件渲染前的初始化过程(参考黄轶老师的源码分析)
> setup 启动函数，它是 Composition API 逻辑组织的入口。

模板中引用到的变量和函数是setup的返回值，那么它是怎么和setup返回值建立联系的呢？

这个过程主要发生在渲染vnode的过程中：挂载组件
```
const mountComponent = (initialVNode, container, anchor, parentComponent, parentSuspense, isSVG, optimized) => {
  // 创建组件实例
  const instance = (initialVNode.component = createComponentInstance(initialVNode, parentComponent, parentSuspense))
  // 设置组件实例
  setupComponent(instance)
  // 设置并运行带副作用的渲染函数
  setupRenderEffect(instance, initialVNode, container, anchor, parentSuspense, isSVG, optimized)
}
```
其中的创建组件实例和设置组件实例的过程就完成了我们上面所说的模板中的变量和方法与setup返回值建立联系的过程，下面先看下createComponentInstance的实现：
 
