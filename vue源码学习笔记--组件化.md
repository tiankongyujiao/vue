### vue源码学习笔记--组件化
> 在实际开发中，vue项目通常是由很多组件拼装一起组成的，每个单文件组件都有模版（DOM），js，css（css可以使用scoped限制在当前单文件组件中起作用）组成。  

一个单文件组件是怎么和其他单文件组件联系起来的呢，单文件组件又是怎么创建虚拟vnode以及如何patch，下面来分析这个过程：

一个简单的vue-cli生成的项目看着简单，其实vue源码做了很多的事情：new Vue生成实例，合并配置，一系列初始化工作以后，挂载调用$mount方法，调用update函数，update的参数render函数首先生成vnode，然后update的时候调用patch方法生成真实dom，这样避免了直接操作dom，提高了性能，vue源码底层还是做了很多事情的。  

```
import Vue from 'vue'
import App from './App.vue'

var app = new Vue({
  el: '#app',
  // 这里的 h 是 createElement 方法
  render: h => h(App)
})
```
#### 一.调用createElement方法生成组件VNode
在createElement调用_createElement方法生成虚拟dom的时候有个对tag标签的判断，如果是个普通的dom（即tab是个html标签（字符串））比如div，会创建一个普通的VNode节点，如果这时候我们的tag是个组件，那么tag就是一个Component类型的对象（上面我们传入的是一个App对象），这时候直接会调用createComponent方法创建一个组件VNode。 
createComponent方法定义在‘src/core/vdom/create-component.js’中，这个方法主要来看三点：
1. 构造子类构造函数：
省略其他不相关的逻辑，如下：  
```
export function createComponent (
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  // ...
  const baseCtor = context.$options._base
  // plain options object: turn it into a constructor
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor)
  }
  // ...
}
```
我们在编写组件的时候，通常是创建一个普通的对象，如:
```
import HelloWorld from './components/HelloWorld'

export default {
  name: 'app',
  components: {
    HelloWorld
  }
}
```
这里export的是一个对象，所以createComponent的代码逻辑会走到Ctor = baseCtor.extend(Ctor)这一步，这里的baseCtor就是指代Vue，所以实际是调用了Vue.extend继承方法，这个方法是定义在‘src/core/global-api/extend.js’中：  
这个继承方法的作用就是构造Vue的一个子类，这个方法传入的对象就是组件对象，比如上面的App（在单文件组件中export default导出的那个对象），这里继承的方法是使用了Object.create()函数（一种经典的原型继承的方式）：
```
 Vue.extend = function (extendOptions: Object): Function {
  // ... 
  const Super = this
  // ...
  const Sub = function VueComponent (options) {
    this._init(options)
  }
  Sub.prototype = Object.create(Super.prototype)
  Sub.prototype.constructor = Sub
  Sub.cid = cid++
  Sub.options = mergeOptions(
    Super.options,
    extendOptions
  )
  // ...
 }
```
然后对Sub扩展了options，添加全局的API，并对配置中的props和computed做了初始化工作，并且最后对这个Sub构造函数做了缓存，避免多次执行 Vue.extend 的时候对同一个子组件重复构造。   
当我们实例化一个Sub的时候，就会调用Sub构造函数中的this._init(options)方法，即Vue原型上的_init方法，再次走到了Vue实例化的初始逻辑。
2. 接下来是安装组件钩子函数  
```
export function createComponent (
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  // ...
  // install component management hooks onto the placeholder node
  installComponentHooks(data)
  // ...
}
```
在初始化一个组件Component类型的VNode的过程中实现了几个钩子函数：init，prepatch，insert，destroy。   
整个 installComponentHooks 的过程就是把 componentVNodeHooks 的钩子函数合并到 data.hook 中，然后再执行VNode的patch过程中执行相应的钩子函数。这里要注意的是合并策略，在合并过程中，如果某个时机的钩子已经存在 data.hook 中，那么通过执行 mergeHook 函数做合并，这个逻辑很简单，就是在最终执行的时候，依次执行这两个钩子函数即可。   
3. 实例化VNode
在createComponent的最后实例化VNode：
```
export function createComponent (
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  // ...
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )
  // ... 
}
```
这里可以看到组件的VNode是没有parent的，是undefined，这点和普通的元素节点VNode不同，这点很关键，后面的patch过程也会提到。
#### 二.创建完组件VNode，调用vm.__patch__把VNode转换成真是的DOM   

patch的过程会调用createElm创建元素节点，createElm方法定义在‘src/core/vdom/patch.js’中：
```
function createElm (
  vnode,
  insertedVnodeQueue,
  parentElm,
  refElm,
  nested,
  ownerArray,
  index
) {
  // ...
  if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
    return
  }
  // ...
}
```

