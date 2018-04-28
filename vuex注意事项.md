#### 1.在 Vuex 模块化中的state,getters,mutations,actions
state 是唯一会根据组合时模块的别名来添加层级的，后面的 getters、mutations 以及 actions 都是直接合并在 store 下。    
（1）例如，访问模块 a 中的 state，要通过 store.state.a，访问根 store 上申明的 state，依然是通过 store.state.xxx 直接访问    
（2）与 state 不同的是，不同模块的 getters 会直接合并在 store.getters 下；getters 的回调函数所接收的前两个参数，模块化后需要用到第三个参数——rootState。参数：
    1️⃣. state，模块中的 state 仅为模块自身中的 state；2️⃣. getters，等同于 store.getters；3️⃣. rootState，全局 state。 
    通过 rootState，模块中的 getters 就可以引用别的模块中的 state 了，十分方便。    
    由于 getters 不区分模块，所以不同模块中的 getters 如果重名，Vuex 会报出 'duplicate getter key: [重复的getter名]' 错误    
（3）mutations 与 getters 类似，不同模块的 mutation 均可以通过 store.commit 直接触发。    
mutation 的回调函数中只接收唯一的参数——当前模块的 state。如果不同模块中有同名的 mutation，Vuex 不会报错，通过 store.commit 调用，会依次触发所有同名 mutation。    
（4）与 mutations 类似，不同模块的 actions 均可以通过 store.dispatch 直接触发。  

action 的回调函数接收一个 context 上下文参数，context 包含：1. state、2. rootState、3. getters、4. mutations、5. actions 五个属性，为了简便可以在参数中解构。

在 action 中可以通过 context.commit 跨模块调用 mutation，同时一个模块的 action 也可以调用其他模块的 action。

同样的，当不同模块中有同名 action 时，通过 store.dispatch 调用，会依次触发所有同名 actions。

最后有一点要注意的是，将 store 中的 state 绑定到 Vue 组件中的 computed 计算属性后，对 state 进行更改需要通过 mutation 或者 action，在 Vue 组件中直接进行赋值 (this.myState = 'ABC') 是不会生效的。

除非在computed计算属性中设置get和set可以使 this.myState = 'ABC'生效，使用mapState的方式不可以：
```
computed: 
      // mapState({
      //   count: state => state.count
      // })
     {
       count: {
         get: function () {
             return this.$store.state.a.count
         },
         set: function (newValue) {
             this.$store.state.a.count = newValue
         }
     }
     }
   
```
但是一般不在组件中直接修改从vuex计算出来的属性，修改store中的值，要通过dispatch => action，在action中commit => mutation这种方式，如果是同步的，可以直接commit => mutation，如果有异步，要先dispatch => action，再commit => mutation，因为mutation中放的都是同步的，action可以放异步的，直接通过 this.myState = 'ABC’这种方式修改，也是异步。异步不能确保修改某一个字段的先后顺序。
