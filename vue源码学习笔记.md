vue源码学习笔记
+ vue通过rollup构建
+ 我们平时用的vue单文件结合webpack使用时一般用的是Vue Runtime Only版本，而不是Runtime + Compiler版本，单文件组件里能写template模板，是因为使用了vue-loader，在编译阶段就把浏览器不认识的vue编译成了JavaScript，把template模板已经编译成了render函数，如果不适用webpack模式的vue，没有vue-loader做预编译，则需要使用Runtime + Compiler版本的文件。
##### vue的Runtime + Compiler版本的入口文件在：src/platforms/web/entry-runtime-with-compiler.js
+ 这个文件最终导出了Vue，Vue是import（'./runtime/index'）而来，随后在Vue上面有挂载了$mount方法（这个$mount方法是跟带编译器相关的方法)。
+ 接下来看下import的来源，'./runtime/index'文件，
+ 首先这个文件也是导出了Vue，这里的Vue也是import（'core/index'）而来，
+ './runtime/index'：这个文件里定义了一些全局的配置以及__patch__方法，以及$mount方法（这个$mount方法是统一方法)，
+ 然后看下'core/index'这个文件，这个文件也是导出了Vue，这里的Vue又是import（'./instance/index'）而来，
##### 定义vue class的地方: 

