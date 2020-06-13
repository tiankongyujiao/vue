vue源码学习笔记
+ vue通过rollup构建
+ 我们平时用的vue单文件结合webpack使用时一般用的是Vue Runtime Only版本，而不是Runtime + Compiler版本，单文件组件里能写template模板，是因为使用了vue-loader，在编译阶段就把浏览器不认识的vue编译成了JavaScript，把template模板已经编译成了render函数，如果不适用webpack模式的vue，没有vue-loader做预编译，则需要使用Runtime + Compiler版本的文件。
##### 定义vue class的地方:

