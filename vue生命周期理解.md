#### vue生命周期理解
当有父子组件时，vue生命周期钩子函数的渲染顺序：
```
// 渲染更新过程
父组件 beforeCreate()：全局的配置和根实例的一些属性初始化，此时data属性为undefined，没有可供操作的数据。
        ↓
父组件 created()：数据代理和动态数据绑定(data observer)，属性和方法的运算， watch/event 事件回调。然而，挂载阶段还没开始，$el 属性目前不可见
        ↓
父组件 beforeMount()：此时已经开始执行模板解析函数，但还没有将$el元素挂载页面，页面视图因此也未更新
        ↓
子组件 beforeCreate()
        ↓
子组件 created()
        ↓
子组件 beforeMount()
        ↓
子组件 mounted()：
        ↓
父组件 mounted()：子组件已经挂载到父组件上，并且父组件随之挂载到页面中。
        ↓
父组件 beforeUpdate()
        ↓
子组件 beforeUpdate
        ↓
子组件 updated()
        ↓
父组件 updated()


// 消亡过程：
父组件 beforeDestroy()： beforeDestroy钩子函数在实例销毁之前调用。在这一步，实例仍然完全可用。
        ↓
子组件beforeDestory()
        ↓
子组件 destoryed()
        ↓
父组件 destoryed() ： Vue 实例销毁后调用。调用后，Vue 实例指示的所有东西都会解绑定，所有的事件监听器会被移除，所有的子实例也会被销毁（也就是说子组件也会触发相应的函数）。这里的销毁并不指代'抹去'，而是表示'解绑'。

        
```
##### 知识点：
1. 在`created()`钩子函数中：已经可以获得data数据，此时可以调用ajax函数修改data，为页面数据做准备；    
2. 在`mounted()`钩子函数中：页面元素已经渲染完成，但有时存在一些异常情况（比如父子组件，在子组件的mounted()函数中修改了数据，在父组件的mounted()函数中获取视图中的页面元素，此时页面中的元素还是`修改之前`的，但是如果在父组件的mounted中打印子组件修改的数据，是修改之后的，因为子组件的mounted先于父组件的mounted()执行）,所以为了保险起见，可以在mounted()中获取或者修改页面元素时可以放在`this.$nextTick(function(){DOM操作})`中。    
3. 在使用vue-router时有时需要使用`<keep-alive></keep-alive>`来缓存组件状态，这个时候created钩子就不会被重复调用了，如果我们的子组件需要在每次加载或切换状态的时候进行某些操作，可以使用`activated`钩子触发。    
4. 虽然updated函数会在数据变化时被触发，但却不能准确的判断是那个属性值被改变，所以在实际情况中用computed或watch函数来监听属性的变化，并做一些其他的操作。    
5. 所有的生命周期钩子自动绑定 this 上下文到实例中，所以不能使用箭头函数来定义一个生命周期方法 (例如 created: () => this.fetchTodos()，这时导致this指向父级    
6. 其实在beforeMount阶段之后、Mounted阶段之前，数据已经被加载到视图上了，即$el元素被挂载到页面时触发了视图的更新。
