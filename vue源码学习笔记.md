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




