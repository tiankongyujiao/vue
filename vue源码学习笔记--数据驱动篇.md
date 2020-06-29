**vue源码学习笔记--数据驱动篇**
##### 构建
+ vue通过rollup构建
+ 我们平时用的vue单文件结合webpack使用时一般用的是Vue Runtime Only版本，而不是Runtime + Compiler版本，单文件组件里能写template模板，是因为使用了vue-loader，在编译阶段就把浏览器不认识的vue编译成了JavaScript，把template模板已经编译成了render函数，如果不使用webpack模式的vue，没有vue-loader做预编译，则需要使用Runtime + Compiler版本。
##### vue的Runtime + Compiler版本的入口文件在：src/platforms/web/entry-runtime-with-compiler.js
+ 'src/platforms/web/entry-runtime-with-compiler.js': 导出了Vue，Vue是import（'src/platforms/web/runtime/index.js'）而来，随后在Vue上面有挂载了$mount方法（这个$mount方法是跟带编译器相关的方法)。
+ 'src/platforms/web/runtime/index.js': 导出了Vue，这里的Vue也是import（'src/core/index.js'）而来，然后定义了一些全局的配置以及__patch__方法，以及$mount方法（这个$mount方法是统一方法)，
+ 'src/core/index.js'：也是导出了Vue，这里的Vue又是import（'src/core/instance/index.js'）而来，然后通过initGlobalAPI方法初始化了一些全局的api，initGlobalAPI方法定义在'src/core/global-api/index.js'中，initGlobalAPI中定义了全局的config，以及set，delete，nextTick全局静态方法，在最后还定义了initUse(Vue)，initMixin(Vue)，initExtend(Vue)，initAssetRegisters(Vue)这些静态方法。
+ 最终在'src/core/instance/index.js'文件中找到了Vue的定义，终于找到你了。。
##### 'src/core/instance/index.js'
+ Vue实际就是一个function，而且Vue必须通过new的方式初始化，不然会报一个错误：vue必须通过new来调用，
+ 在这个文件里定义了一堆mixin方法，这也是vue不适用class来定义的原因。这些mixin包括：initMixin(Vue),stateMixin(Vue),eventsMixin(Vue),lifecycleMixin(Vue),renderMixin(Vue)
+ initMixin(Vue)的定义是在'src/core/instance/init.js'中, initMixin实际上是往Vue的原型链prototype上挂了一个_init方法。
+ 每个mixin都是往原型上汇入定义的一些原型链的方法。
>从上面可以看到'src/platforms/web/entry-runtime-with-compiler.js'入口文件定义了带编译版本的$mount方法，而后在'src/platforms/web/runtime/index.js'中定义了通用的$mount方法，带编译的$mount最终也会调用通用的$mount。在'src/core/index.js'中主要是定义了一些全局的API，在'src/core/instance/index.js'定义了Vue这个类，并挂在了一系列的原型方法。
### 数据驱动
数据驱动是指视图由数据驱动生成，我们对视图的修改不会直接操作DOM，而是通过直接修改数据，相比于传统的使用jquery或者原生js操作DOM大大简化了代码量，特别是当交互复杂的时候，只关心数据的修改会让代码逻辑变得非常清晰，因为DOM变成了数据的映射，我们所有的逻辑都是对数据的修改，而不触碰DOM，这样的代码非常利于维护。
##### new Vue()
new Vue方法里执行了_init()方法，这个_init()方法是当前文件initMixin(Vue)方法中定义的，initMixin(Vue)这个方法又是在'src/core/instance/init.js'中定义，然后我们来看_init这个方法：
+ _init(options)做了一堆的初始化工作，比如定义_uid，合并options（把传入的options最终merge到$options上，所以可以通过this.$options.el访问到我们代码中定义的el，通过this.$options.data访问到我们代码中定义的data），
+ 接下来定义了一堆初始化的函数，比如initLifecycle(vm)，initEvents(vm)，initRender(vm)，callHook(vm, 'beforeCreate')，initInjections(vm)，initState(vm)，initProvide(vm)，callHook(vm, 'created')，
+ 最后判断我们的vm.$options.el是不是存在，如果存在会调用$mount方法做挂载。
+ 其中initState(vm)挂载了data,props,methods...
+ data会被代理到_data上(通过这行代码实现data = vm._data = typeof data === 'function' ? getData(data, vm) : data || {})，我们访问this.message时实际访问的是this._data.message(通过proxy实现)，但我们在实际开发中不使用_data,因为下划线开头的默认都是私有属性。
##### Vue实例挂载的实现 $mount
+ 上面我们提到过在入口文件'src/platforms/web/entry-runtime-with-compiler.js'和'src/platforms/web/runtime/index.js'（统一$mount）文件中都定义了$mount，入口文件定义的是跟编译相关的$mount，在编译相关的$mount方法里最终会调用统一的$mount方法，编译相关的$mount会在调用统一的$mount前把template编译成render函数，然后调用统一的$mount(这个方法只认render函数)。
+ 在统一的$mount()（'src/platforms/web/runtime/index.js'）中，会调用*mountComponent*方法，mountComponent方法定义在'src/core/instance/lifecycle.js'中。
+ mountComponent(): 首先缓存el（vm.$el = el）,然后判断是否存在render函数（template,el编译成的render函数或者手写的render函数），如果不存在就生成一个空的vnode赋值给render函数，并如果在开发环境根据情况报一系列的警告。
+ 然后再mountComponent()最后调用updateComponent()
+ updateComponent(): 这个方法的执行是在new Watcher（渲染watcher）中调用（this.getter,这个getter就是我们传入new Watcher中的updateComponent()）,这样就执行了updateComponent()，也就执行了*vm._update(vm._render(),hydrating)*,vm._update和vm._render函数是我们挂载到真实DOM要用到的两个函数。vm._render是生成一个vnode，然后调用vm._update把vnode传入，这个后面会重点分析。
##### vm._render
+ 定义在'src/core/instance/render.js'中，定义在renderMixin()方法中的，renderMixin()方法在'src/core/instance/index.js'中调用一些列mixin方法中的其中一个，renderMixin除了在原型上挂载了_render方法，还挂载了$nextTick方法。
+ _render最终返回的是一个vnode。
+ _render函数内首先vm.$options.render赋值给render(const { render, _parentVnode } = vm.$options)，然后调用render，这个render可以是手写的render，也可以是通过编译生成的函数
+ vnode = render.call(vm._renderProxy, vm.$createElement): vm._renderProxy在生产环境其实就是vm，在开发环境就是一个proxy对象，vm.$createElement是在initRender函数中定义，initRender函数是在_init函数中初始化一堆函数中的其中一个。vm.$createElement就是我们手写render函数时传入的那个createElement或者是h形参，形参可以自定义为任意变量，都代表了vm.$createElement。
##### 虚拟DOM
+ Virtual DOM 就是用一个原生的 JS 对象去描述一个 DOM 节点，所以它比创建一个 DOM 的代价要小很多。
+ Virtual DOM 是对真实DOM的一种抽象描述，它的核心定义包括标签名，数据，子节点，键值等。
+ VNode 只是用来映射到真实DOM的渲染，不需要包含操作DOM的方法，因此它非常轻量和简单。
+ Virtual DOM 除了它的数据结构的定义，映射到真实的DOM实际上要经历VNode的create,diff,patch等过程。VNode的create是我们之前提到的，vm.$createElement方法创建的，下面分析这个方法。
##### createElement
+ createElement方法是在'src/core/vdom/create-element.js'中定义的，在这个方法中对参数进行了处理，就是如果没有传入data的情况，把后面的参数分别往前移动一个，处理完了以后调用了_createElement函数。
+ _createElement：真正创建VNode的函数。 
+ _createElement中最主要的两个方法normalizeChildren和simpleNormalizeChildren方法，其中simpleNormalizeChildren是经过编译生成的render使用的，normalizeChildren是手写render使用的。
### Vue.prototype._update
在updateComponent函数中执行的 **vm._update(vm._render(), hydrating)**，vm._render()执行后得到的是vnode，返回后传给vm._update方法把vnode渲染成真实的DOM。
+ _update 的核心就是调用 vm.__patch__方法，这个方法在不同的平台（web和weex）定义是不一样的，在web中是定义在‘src/platforms/web/runtime/index.js’中,这里利用了函数柯里化的技巧把平台化的差异在“src\core\vdom\patch.js”中抹平，而不用每次使用时都用if...else...区分平台，'src\core\vdom\patch.js'这个文件最后返回了patch方法，就是我们实际调用的__patch__方法。
+ 核心方法createElm就是把vnode挂载到真实的dom，里层的createChildren会递归调用createElm，一层一层传，后面会调用insert方法插入到dom，insert是从子开始插入到父，一层层执行下去，直到最顶层的父级插入到body。整个插入顺序就是先子后父。

