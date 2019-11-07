#### vuex的简单理解
vuex是vue的状态管理模式。    

通常vue的单页面组件都有自己的data，在没有vuex之前我们把交互的数据都放在data里面，这样使得多组件使用的状态（data）传值变得复杂、不好维护，于是出现了vuex。     

vuex管理统一的状态，在mutation中修改状态，不能直接修改，必须显示的通过store.commit(mutation)来提交状态，且mutation里面都是同步操作，不能有异步操作，异步操作放在store的action中去提交，在组件中通过store.dispatch(action)来触发action，然后在action中通过store.commit(mutation)来触发mutation。  

并不是所有的状态都要放到vuex中，需要根据具体情况来定，一般公用的一些状态，且不经常改动的状态可以放到store中管理，需要改动的异步获取的数据需要通过触发aciton来获取；一个组件内使用的状态，直接写在组件的data里面就好了。  


vuex包含的知识点：    
1. state：状态存放的地方     
2. getters：计算state中的data，和组件中的computed类似    
3. mutations：更改 Vuex 的 store 中的状态的唯一方法是提交 mutation：更改状态的地方    
4. actions：类似于 mutation，但是可以进行异步操作，所有需要进行异步操作的状态要现在action中进行异步操作，然后在aciton触发mutation来更改状态    
5. modules：vuex中的模块，避免vuex的store过于臃肿庞大，可以为项目中每个不同的模块分别定义一个store的模块，即这里的module，module和统一的store中的属性都一样，可以有`state`, `getters`,`mutations`,`actions`，还一个有个命名空间属性namespaced，namespaced为true，表示启用命名空间。一般使用modules时需要加上namespaced，在组件中使用模块中的store时更加清晰，知道来源于哪个模块。

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
  SET_NAME: (state, name) => {
    state.name = name
  },
  SET_AGE: (state, {age}){ // 使用es6的解构赋值
    state.age = age
  }
}
const actions = { // actions方法在单个vue组件中通过store.dispatch('moduleA/setName')来触发，要加上moduleA，因为使用了命名空间
  setName ({ commit }, name) {
    commit('SET_NAME', name)
  },
  setAge ({ commit }){
    setTimeout(function(){ // 异步函数：或者ajax请求
      commit('SET_AGE', {age: '12'}) // 使用es6的解构赋值，这里age随便取了一个值，如果是ajax异步操作，则是获取到的数据
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
