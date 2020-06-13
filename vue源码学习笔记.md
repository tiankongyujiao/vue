vue源码学习笔记
+ vue通过rollup构建
+ 我们平时用的vue单文件结合webpack使用时一般用的是Vue Runtime Only版本，而不是Runtime + Compiler版本，单文件组件里能写template模板，是因为使用了vue-loader，在编译阶段就把浏览器不认识的vue编译成了JavaScript，把template模板已经编译成了render函数，如果不适用webpack模式的vue，没有vue-loader做预编译，则需要使用Runtime + Compiler版本的文件。
##### vue的Runtime + Compiler版本的入口文件在：src/platforms/web/entry-runtime-with-compiler.js
+ 'src/platforms/web/entry-runtime-with-compiler.js': 导出了Vue，Vue是import（'src/platforms/web/runtime/index.js'）而来，随后在Vue上面有挂载了$mount方法（这个$mount方法是跟带编译器相关的方法)。
+ 'src/platforms/web/runtime/index.js': 导出了Vue，这里的Vue也是import（'src/core/index.js'）而来，然后定义了一些全局的配置以及__patch__方法，以及$mount方法（这个$mount方法是统一方法)，
+ 'src/core/index.js'：也是导出了Vue，这里的Vue又是import（'src/core/instance/index.js'）而来，然后通过initGlobalAPI方法初始化了一些全局的api，initGlobalAPI方法定义在'src/core/global-api/index.js'中，initGlobalAPI中定义了全局的config，以及set，delete，nextTick全局静态方法，在最后还一定initUse(Vue)方法，initMixin(Vue)，initExtend(Vue)，initAssetRegisters(Vue)。
+ 最终在'src/core/instance/index.js'文件中找到了Vue的定义，终于找到你了。。
##### 'src/core/instance/index.js'
+ Vue实际就是一个function，而且Vue必须通过new的方式初始化，不然会报一个错误：vue必须通过new来调用，
+ 在这个文件里定义了一堆mixin方法，这也是vue不适用class来定义的原因。这些mixin包括：initMixin(Vue),stateMixin(Vue),eventsMixin(Vue),lifecycleMixin(Vue),renderMixin(Vue)
+ initMixin(Vue)的定义是在'src/core/instance/init.js'中, initMixin实际上是往Vue的原型链prototype上挂了一个_init方法。
+ 每个mixin都是往原型上汇入定义的一些原型链的方法。
### 数据驱动
数据驱动是指视图由数据驱动生成，我们对视图的修改不会直接操作DOM，而是通过直接修改数据，相比于传统的使用jquery或者原生js操作DOM大大简化了代码量，特别是当交互复杂的时候，之关系数据的修改会让代码逻辑变得非常清晰，因为DOM变成了数据的映射，我们所有的逻辑都是对数据的修改，而不触碰DOM，这样的代码非常利于维护。
##### new Vue()
new Vue方法里执行了_init()方法，这个_init()方法是我们上面提到的'src/core/instance/index.js'文件中的initMixin方法中定义的，这个方法又是在'src/core/instance/init.js'中定义，然后我们来看_init这个方法：
+ 做了一堆的初始化的工作，比如定义_uid，合并options（把传入的options最终merge到$options上，所以可以通过this.$options.el访问到我们代码中定义的el，通过this.$options.data访问到我们代码中定义的data），
+ 接下来定义了一堆初始化的函数，比如initLifecycle(vm)，initEvents(vm)，initRender(vm)，callHook(vm, 'beforeCreate')，initInjections(vm)，initState(vm)，initProvide(vm)，callHook(vm, 'created')，
+ 最后判断我们的vm.$options.el是不是存在，如果存在会调用$mount方法做挂载。



