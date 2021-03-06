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
在createElm方法中首先会调用createComponent方法：
```
function createComponent(vnode, insertedVnodeQueue, parentElm, refElm) {
  let i = vnode.data
  if (isDef(i)) {
      const isReactivated = isDef(vnode.componentInstance) && i.keepAlive
      if (isDef(i = i.hook) && isDef(i = i.init)) {
          i(vnode, false /* hydrating */ )
      }
      // after calling the init hook, if the vnode is a child component
      // it should've created a child instance and mounted it. the child
      // component also has set the placeholder vnode's elm.
      // in that case we can just return the element and be done.
      if (isDef(vnode.componentInstance)) {
          initComponent(vnode, insertedVnodeQueue)
          insert(parentElm, vnode.elm, refElm)
          if (isTrue(isReactivated)) {
              reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)
          }
          return true
      }
  }
}
 ```
注意这个createComponent方法要和创建VNode的createComponent方法区分开，那个是为了创建VNode，以及构造子类用的，这个是有了子类以后调用子类的_init方法用的。   
我们在创建VNode的时候传入的data参数，首先判断是否有data，如果是正常初始化子类的话是有data的且有hooks钩子，然后调用钩子的init方法，即上文中我们提到的installComponentHooks(data)挂载到子类上的一些钩子，这里我们调用的是init钩子，这个钩子定义在创建子类VNode的‘src/core/vdom/create-component.js’中：
```
init (vnode: VNodeWithData, hydrating: boolean): ?boolean {
  if (
    vnode.componentInstance &&
    !vnode.componentInstance._isDestroyed &&
    vnode.data.keepAlive
  ) {
    // kept-alive components, treat as a patch
    const mountedNode: any = vnode // work around flow
    componentVNodeHooks.prepatch(mountedNode, mountedNode)
  } else {
    const child = vnode.componentInstance = createComponentInstanceForVnode(
      vnode,
      activeInstance
    )
    child.$mount(hydrating ? vnode.elm : undefined, hydrating)
  }
  }
```
这里我们会走到上面代码中的else逻辑，调用了createComponentInstanceForVnode实例化一个Vue的实例，然后通过$mount挂载子组件，下面分析这个过程：   
1. 调用 createComponentInstanceForVnode 创建一个 Vue 的实例
```
export function createComponentInstanceForVnode (
  vnode: any, // we know it's MountedComponentVNode but flow doesn't
  parent: any, // activeInstance in lifecycle state
): Component {
  const options: InternalComponentOptions = {
    _isComponent: true,
    _parentVnode: vnode,
    parent
  }
  // check inline-template render functions
  const inlineTemplate = vnode.data.inlineTemplate
  if (isDef(inlineTemplate)) {
    options.render = inlineTemplate.render
    options.staticRenderFns = inlineTemplate.staticRenderFns
  }
  return new vnode.componentOptions.Ctor(options)
}
```
+ 首先构建组件的参数options，然后执行new vnode.componentOptions.Ctor(options)创建一个Vue实例，并返回这个实例。这里的vnode.componentOptions.Ctor对应的就是子组件的构造函数，因为上面我们提到的实例化VNode的时候传入的第七个参数是一个解构对象，解构对象的第一个参数就是子组件的构造函数Ctor，VNode的第七个参数名字叫componentOptions，所以这时候我们通过vnode.componentOptions.Ctor获取的自然就是子组件的构造函数，即继承Vue的子构造器Sub，相当于new Sub(options)
+ 参数options对象的isComponent为true代表的是一个组件，parent表示当前激活的组件实例
+ 以上过程就完成了子组件的实例化，执行‘new Sub(options)’会调用Sub构造函数里的_init()方法初始化。
+ _init和普通节点不同的是多了参数的合并，把通过createComponentInstanceForVnode函数传入的参数合并到vm.$options中。
+ 在_init方法的最后并不会调用这里的$mount方法挂载组件，因为组件没有el，还记得我们之前创建组件实例以后，在createComponent的init钩子的最后调用了组件的child.$mount(hydrating ? vnode.elm : undefined, hydrating)方法，组件自己接管了$mount挂载，这里相当于执行了child.$mount(undefined, false)，它最终会调用mountComponent方法，最终调用vm._update(vm._render(), hydratin)，首先会调用vm._render()方法：
```
Vue.prototype._render = function (): VNode {
  const vm: Component = this
  const { render, _parentVnode } = vm.$options

  
  // set parent vnode. this allows render functions to have access
  // to the data on the placeholder node.
  vm.$vnode = _parentVnode
  // render self
  let vnode
  try {
    vnode = render.call(vm._renderProxy, vm.$createElement)
  } catch (e) {
    // ...
  }
  // set parent
  vnode.parent = _parentVnode
  return vnode
}
```
这里只保留关键代码，这里的 _parentVnode 就是当前组件的父 VNode，而 render 函数生成的 vnode 是当前组件的渲染 vnode，vnode 的 parent 指向了 _parentVnode，也就是 vm.$vnode，它们是一种父子的关系。  
这个调用render.call(vm._renderProxy, vm.$createElement)生成VNode，这里的这个render函数实际是通过webpack的vue-loader函数把单文件组件自动生成了一个render函数，这时候它的tag最外层的就是一个像div这样的html标签，即一个普通的节点，如果div中还有子组件，则会重复createElement函数中的createComponent函数，生成一个组件VNode，然后再去pathc生成真实dom，依此循环往下推，就构成了一个DOM树。  
在执行完了render方法生成vnode以后会调用vm._update去渲染vnode：
```
export let activeInstance: any = null
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  const prevEl = vm.$el
  const prevVnode = vm._vnode
  const prevActiveInstance = activeInstance
  activeInstance = vm
  vm._vnode = vnode
  // Vue.prototype.__patch__ is injected in entry points
  // based on the rendering backend used.
  if (!prevVnode) {
    // initial render
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
  } else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  activeInstance = prevActiveInstance
  // update __vue__ reference
  if (prevEl) {
    prevEl.__vue__ = null
  }
  if (vm.$el) {
    vm.$el.__vue__ = vm
  }
  // if parent is an HOC, update its $el as well
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el
  }
  // updated hook is called by the scheduler to ensure that children are
  // updated in a parent's updated hook.
}
```
_update 过程中有几个关键的代码，首先 vm._vnode = vnode 的逻辑，这个 vnode 是通过 vm._render() 返回的组件渲染 VNode，vm._vnode 和 vm.$vnode 的关系就是一种父子关系，用代码表达就是 vm._vnode.parent === vm.$vnode，还有一段比较有意思的代码：
```
export let activeInstance: any = null
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    // ...
    const prevActiveInstance = activeInstance
    activeInstance = vm
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    activeInstance = prevActiveInstance
    // ...
}
```
这里会把当前激活的activeInstance赋值给prevActiveInstance，然后把vm赋值给activeInstance，即当前的vm成了当前激活的实例，这个activeInstance是在lifecycle模块定义的全局的变量，在‘src/core/instance/lifecycle.js’中有这么一行：*export let activeInstance: any = null*。  
我们在createComponentInstanceForVnode时从lifecycle中获取activeInstanc作为createComponentInstanceForVnode的第二个参数，作为子组件的父实例parent，从而建立了父子关系的前提基础。  
因为js是一个单线程，Vue的整个初始化时一个深度遍历的过程，在实例化子组件的过程中，它需要知道当前上下文的Vue实例是什么，并把它作为子组件的父Vue实例。  
之前我们提到在对子组件初始化init的时候回调用initInternalComponent(vm, options)合并options，把parent存储在vm.$options中，并在init中（$mount之前）会调用initLifecycle(vm)方法，这个方法建立了父子关系：
```
export function initLifecycle (vm: Component) {
  const options = vm.$options

  // locate first non-abstract parent
  let parent = options.parent
  if (parent && !options.abstract) {
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent
    }
    parent.$children.push(vm)
  }

  vm.$parent = parent
  // ...
}
```
可以看到 vm.$parent 就是用来保留当前 vm 的父实例，并且通过 parent.$children.push(vm) 来把当前的 vm 存储到父实例的 $children 中。
> 在 vm._update 的过程中，把当前的 vm 赋值给 activeInstance，同时通过 const prevActiveInstance = activeInstance 用 prevActiveInstance 保留上一次的 activeInstance。实际上，prevActiveInstance 和当前的 vm 是一个父子关系，当一个 vm 实例完成它的所有子树的 patch 或者 update 过程后，activeInstance 会回到它的父实例，这样就完美地保证了 createComponentInstanceForVnode 整个深度遍历过程中，我们在实例化子组件的时候能传入当前子组件的父 Vue 实例，并在 _init 的过程中，通过 vm.$parent 把这个父子关系保留。
那么回到 _update，最后就是调用 __patch__ 渲染 VNode 了：
```
vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
 
function patch (oldVnode, vnode, hydrating, removeOnly) {
  // ...
  let isInitialPatch = false
  const insertedVnodeQueue = []

  if (isUndef(oldVnode)) {
    // empty mount (likely as component), create new root element
    isInitialPatch = true
    createElm(vnode, insertedVnodeQueue)
  } else {
    // ...
  }
  // ...
}
```
这里调用createElm没有传入parentElm，只有两个参数，所以对应的 parentElm 是 undefined：
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

  const data = vnode.data
  const children = vnode.children
  const tag = vnode.tag
  if (isDef(tag)) {
    // ...

    vnode.elm = vnode.ns
      ? nodeOps.createElementNS(vnode.ns, tag)
      : nodeOps.createElement(tag, vnode)
    setScope(vnode)

    /* istanbul ignore if */
    if (__WEEX__) {
      // ...
    } else {
      createChildren(vnode, children, insertedVnodeQueue)
      if (isDef(data)) {
        invokeCreateHooks(vnode, insertedVnodeQueue)
      }
      insert(parentElm, vnode.elm, refElm)
    }
    
    // ...
  } else if (isTrue(vnode.isComment)) {
    vnode.elm = nodeOps.createComment(vnode.text)
    insert(parentElm, vnode.elm, refElm)
  } else {
    vnode.elm = nodeOps.createTextNode(vnode.text)
    insert(parentElm, vnode.elm, refElm)
  }
}
```
这里执行createComponent传入的是render生成的组件渲染vnode，如果组件的根节点是一个真实dom，那么vnode也是一个普通的vnode，createComponent会返回false，接着往下执行到createChildren以及insert，由于这里传入的parentElm是一个空的，所以这里的插入不起作用，还记得我们在调用createComponent生成真实dom的时候，有一段代码：
```
function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
  let i = vnode.data
  if (isDef(i)) {
    // ....
    if (isDef(i = i.hook) && isDef(i = i.init)) {
      i(vnode, false /* hydrating */)
    }
    // ...
    if (isDef(vnode.componentInstance)) {
      initComponent(vnode, insertedVnodeQueue)
      insert(parentElm, vnode.elm, refElm)
      if (isTrue(isReactivated)) {
        reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)
      }
      return true
    }
  }
}
```
其实和组件实例自己接管$mount一样，组件实例也自己接管了insert方法，用来插入到父节点。如果patch的过程中又遇到了子组件，那么DOM的插入顺序是先子后父。  
那么到此，一个组件的 VNode 是如何创建、初始化、渲染的过程也就梳理完成了。  



