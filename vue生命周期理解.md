#### vue生命周期理解
当有父子组件时，vue生命周期钩子函数的渲染顺序：
```
父组件 beforeCreate()
        ↓
父组件 created()
        ↓
父组件 beforeMount()
        ↓
子组件 beforeCreate()
        ↓
子组件 created()
        ↓
子组件 beforeMount()
        ↓
子组件 mounted()
        ↓
父组件 mounted()
        ↓
父组件 beforeUpdate()
        ↓
子组件 beforeUpdate
        ↓
子组件 updated()
        ↓
父组件 updated()
        
```
