#### vuex的简单理解
vuex是vue的状态管理模式。    

通常vue的单页面组件都有自己的data，在没有vuex之前我们把交互的数据都放在data里面，这样使得多组件使用的状态（data）传值变得复杂、不好维护，于是出现了vuex。     

vuex管理统一的状态，在mutation中修改状态，不能直接修改，必须显示的通过store.commit(mutation)来提交状态，且mutation里面都是同步操作，不能有异步操作，异步操作放在store的action中去提交，在组件中通过store.dispatch(action)来触发action，然后在action中通过store.commit(mutation)来触发mutation。  

并不是所有的状态都要放到vuex中，需要根据具体情况来定，一般公用的一些状态，且不经常改动的状态可以放到store中管理，需要改动的异步获取的数据需要通过触发aciton来获取；一个组件内使用的状态，直接写在组件的data里面就好了。  


##### vuex包含的知识点：    
1. `state`：状态存放的地方     
2. `getters`：和组件中的computed类似,从 store 中的 state 中派生出的一些状态 
3. `mutations`：更改 Vuex 的 store 中的状态的唯一方法是提交 mutation：更改状态的地方    
4. `actions`：类似于 mutation，但是可以进行异步操作，所有需要进行异步操作的状态要现在action中进行异步操作，然后在aciton触发mutation来更改状态    
5. `modules`：vuex中的模块，避免vuex的store过于臃肿庞大，可以为项目中每个不同的模块分别定义一个store的模块，即这里的module，module和统一的store中的属性都一样，可以有`state`, `getters`,`mutations`,`actions`，还一个有个命名空间属性namespaced，namespaced为true，表示启用命名空间。一般使用modules时需要加上namespaced，在组件中使用模块中的store时更加清晰，知道来源于哪个模块。

```
import Vue from 'vue';
import Vuex from 'vuex';
import moduleA from './modules/moduleA';
import moduleB from './modules/moduleB';
Vue.use(Vuex);
export default new Vuex.Store({
    state: {
        // 可以有公用的state
    },
    mutations: {
        // 可以有公用的mutations
    },
    actions: {
        // 可以有公用的actions
    },
    modules: {
        moduleA,
        moduleB
    }
});


// moduleA.js
const state = { 
  name: '',
  age: ''
}   
const getters = {
}
const mutations = {
  // mutation 都有一个字符串的 事件类型 (type) 和 一个 回调函数 (handler)：这里的事件类型即‘SET_NAME’，回调函数即为对应的函数
  SET_NAME: (state, { name }) => {
    state.name = name
  },
  SET_AGE: (state, {age}){ // 使用es6的解构赋值
    state.age = age
  }
}
const actions = { // actions方法在单个vue组件中通过store.dispatch('moduleA/setName')来触发，要加上moduleA，因为使用了命名空间
  setName ({ commit }, name) {
    commit('SET_NAME', { name: name }) // 这里的name是 mutation 的 载荷，大多数情况是一个对象
    // 还可以以对象风格提交mutation
    // this.$store.commit({ 
    //   type: 'SET_NAME',
    //   name: name
    // })
  },
  setAge ({ commit }){
    setTimeout(function(){ // 异步函数：或者ajax请求
      commit('SET_AGE', {age: '12'}) // 使用es6的解构赋值，这里age随便取了一个值，如果是ajax异步操作，则是获取到的数据
    })
  },
  // 组合 Action ：store.dispatch 可以处理被触发的 action 的处理函数返回的 Promise，并且 store.dispatch 仍旧返回 Promise
  // 然后就可以这样使用：store.dispatch('actionA').then(() => { ... })，实现多个异步操作
  actionA ({ commit }) {
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        commit('someMutation')
        resolve()
      }, 1000)
    })
  }
}
export default{
    namespaced: true,
    state,
    getters,
    actions,
    mutations
}

// moduleB省略

// 单个vue组件的js部分（简化）
export default {
  data () {
    return {
      // 可以有当前组件内部的data，当然可以通过props，函数触发等方式在父子组件之间通信
    }
  },
  methods: {
    // 修改年龄
    changeAge() {
      this.$store.disptch('moduleA/setAge'); 
      // this.$store是在全局注册过的store 
      // var vue = new Vue({
      //   el: '#app',
      //   router,
      //   store,  // Vuex 通过 store 选项，提供了一种机制将状态从根组件“注入”到每一个子组件中（需调用 Vue.use(Vuex)）
      //   i18n,
      //   components: { App },
      //   template: '<App/>'
      // })
    }
  }
}
```
##### vuex中可以使用`mapState`,`mapGetters`,`mapMutations`,`mapActions`来把vuex的store中的状态等信息映射到单文件组件中，如：
```
import { mapState, mapGetters, mapMutations, mapActions } from 'vuex'
export default {
  data () {
    return {
      // 可以有当前组件内部的data，当然可以通过props，函数触发等方式在父子组件之间通信
    }
  },
  computed: {
    // 组件自身的compute属性
    // mapState
    ...mapState({
        alertData: state => state.alertData
    }),
    // mapGetters
    ...mapGetters([
      'doneTodosCount',
      'anotherGetter',
      // ...
    ])
  },
  methods: {
    // 修改年龄
    changeAge() {
      this.$store.disptch('moduleA/setAge');
    },
    // mapMutations
    ...mapMutations([
      'increment', // 将 `this.increment()` 映射为 `this.$store.commit('increment')`

      // `mapMutations` 也支持载荷：
      'incrementBy' // 将 `this.incrementBy(amount)` 映射为 `this.$store.commit('incrementBy', amount)`
    ]),
    ...mapMutations({
      add: 'increment' // 将 `this.add()` 映射为 `this.$store.commit('increment')`
    }),
    // mapActions
    ...mapActions([
      'increment', // 将 `this.increment()` 映射为 `this.$store.dispatch('increment')`

      // `mapActions` 也支持载荷：
      'incrementBy' // 将 `this.incrementBy(amount)` 映射为 `this.$store.dispatch('incrementBy', amount)`
    ]),
    ...mapActions({
      add: 'increment' // 将 `this.add()` 映射为 `this.$store.dispatch('increment')`
    })
  }
}
```
##### 注意点
1. 在state中声明一个空对象，比如：
```
state: {
    info: {}
}
```
在通过异步请求给info对象赋值以后，比如把{name: 'zhangsan', age: 18}赋值给state.info，后续如果再往info对象中添加其他的属性，比如添加一个sex属性为‘female’，那么这个‘female’属性再次修改时不会动态体现到dom结构中，需要手动使用Vue.$set(state.info, 'sex', 'female')方式赋值。    
上面的name和age可以动态更新到dom中。
