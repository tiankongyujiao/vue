### vue源码学习笔记--组件化--组件注册和异步组件
> 组件有内置组件，例如keep-alive,transition,transition-group，我们还可以自定义自己的通用组件或者业务组件，通用组件一般采用全局注册：Vue.component('my-component', { // 选项}),业务组件一般采用局部注册，在单文件组件中直接导入另一个单文件组件，然后在compontents属性上注册引入进来的单文件组建。
