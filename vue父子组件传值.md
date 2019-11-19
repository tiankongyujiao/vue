### 父子组件传值注意事项
##### 1.通过props传值    
```
// 父组件

<button @click="selfEvent">click parent button</button>
<child-component :count="count" @childEvent="selfEvent"></child-component>

import childComponent from './components/childComponent.vue'
export default{
  data() {
    return{
      count: 0
    }
  },
  methods: {
    selfEvent() {
      this.count ++;
    }
  },
  components: {
    childComponent
  }
}
// 子组件childComponent.vue'
<div>
  {{ count }}
  <button @click="changeValue">click child button</button>
</div>

export default {
  data(){
    return {}
  },
  props: ['count'],
  // 或者定义props值的类型
  // props: {
  //   count: Number
  // }
  methods: {
    changeValue() {
      this.$emit('childEvent');
    }
  }
}
```
子组件通过props接收父组件的传值，通过$emit方法触发父组件事件改变传值（不能在子组件修改父组件的值），如果父组件传递给子组件的值是动态的（ajax异步请求获取来的），需要在子组件watch传值，然后赋值给子组件自身的data属性。

##### 2. sync方式
使用sync实现数据的双向绑定，在vue是不允许双向绑定的，可以使用sync变相实现，例如：
```
<div id="example-2">
// 这里的 :message.sync='msg'相当于 v-bind:message.sync='msg' 相当于 v-bind:message="msg" v-on:update:message="msg = $event"
  <text-document :message.sync='msg'>
    <template #h1="{ slotTest = {zjtest: 'enen'}}"><h1>nothing is impossible for a willing heart!{{ slotTest.zjtest }}</h1></template>
    <template v-slot:h2="ai"><h2 slot="h2">{{ ai.lala }}</h2></template>
  </text-document>
</div>
Vue.component('text-document', {
    props: {
      message: String
    },
    data() {
        return {
            lala: 'lallalal',
            test:{
                zjtest: 'ddd'
            },
            test2: undefined
        }
    },
    template: `<div>子组件的值：{{ message }}
        <button @click="clickEvent">change value</button>
        <slot name="h1" v-bind:slotTest="test2"></slot>
        <slot name="h2" v-bind:lala="lala"></slot>
        </div>`,
    methods: {
        clickEvent(){
            this.$emit('update:message', 456);
        }
    }
})
var vm = new Vue({
    el: '#example-2',
    data: {
        msg: '123',
    }
})
```
使用:message.sync='msg'，然后在子组件里使用this.$emit('update:message', 456)，就可以改变父组件的值，父组件无需再写update:message方法
##### 3. 通过$listeners传递事件和$attrs传递属性
`$attrs`：包含了父作用域中不作为prop被识别（且获取）的特性绑定（class和style除外）。当一个组件没有声明任何prop时，这里会包含所有父作用域的绑定（class和style除外），并且可以通过v-bind="$attrs"传入内部组件：在创建高级别的组件的时候非常有用。    
`$listeners`：包含了父作用域中（不含.native修饰器的）v-on事件监听器。它可以通过v-on="$listeners"传入内部组件：在创建高级别的组件的时候非常有用。
##### 4. 使用vue2.6.0新增的 Vue.observable( object )
作用：让一个对象可响应。Vue 内部会用它来处理 data 函数返回的对象。    
返回的对象可以直接用于`渲染函数`和`计算属性`内(试了一下，不能直接写在模板中，如单文件组件的template中，会报错)，并且会在发生改变时触发相应的更新。也可以作为最小化的跨组件状态存储器，用于简单的场景：
```
const state = Vue.observable({ count: 0 })
const Demo = {
  render(h) {
    return h('button', {
      on: { click: () => { state.count++ }}
    }, `count is: ${state.count}`)
  }
}
// 或者用于计算属性中
```
##### 5. 通过provider/inject
简单的来说就是在父组件中通过provider来提供变量，然后在子组件中通过inject来注入变量。
```
// 父组件
<template>
  <div>
    <childOne></childOne>
  </div>
</template>

<script>
  import childOne from '../components/child'
  export default {
    name: "Parent",
    provide: {
      test: "zjtest"
    },
    components:{
      childOne
    }
  }
</script>

// 子组件
<template>
  <div>
    {{demo}}
    <childtwo></childtwo>
  </div>
</template>

<script>
  import childTwo from './ChildTwo'
  export default {
    name: "childOne",
    inject: ['test'],
    data() {
      return {
        demo: this.test
      }
    },
    components: {
      childTwo
    }
  }
</script>

<template>
  <div>
    {{demo}}
  </div>
</template>
// 子组件的子组件
<script>
  export default {
    name: "childTwo",
    inject: ['test'],
    data() {
      return {
        demo: this.test
      }
    }
  }
</script>
```
使用provider/inject时，不管父子关系层级有多深，只要父组件或者父组件的父组件...以此类推的组件定义了provider，都可以在当前组件中使用inject获得provider中提供的属性。
##### 6. eventBus
作用：主要是现实途径是在要相互通信的兄弟组件之中，都引入一个新的vue实例，然后通过分别调用这个实例的事件触发和监听来实现通信和参数传递。    
使用方法：// 转自https://segmentfault.com/a/1190000013636153  
比如，我们这里有三个组件，main.vue、click.vue、show.vue。click和show是父组件main下的兄弟组件，而且click是通过v-for在父组件中遍历在了多个列表项中。这里要实现，click组件中触发点击事件后，由show组件将点击的是哪个dom元素console出来。    
首先，我们给click组件添加点击事件
```
<div class="click" @click.stop.prevent="doClick($event)"></div>  
```
想要在doClick()方法中，实现对show组件的通信，我们需要新建一个js文件，来创建出我们的eventBus，我们把它命名为bus.js
```
import Vue from 'vue';  
export default new Vue(); 
```
这样我们就创建了一个新的vue实例。接下来我们在click组件和show组件中import它。
```
import Bus from 'common/js/bus.js';  
```
接下来，我们在doClick方法中，来触发一个事件：
```
methods: {  
  addCart(event) {  
    Bus.$emit('getTarget', event.target);   
  }  
}  
```
这里我们在click组件中每次点击，都会在bus中触发这个名为'getTarget'的事件，并将点击事件的event.target顺着事件传递出去。

接着，我们要在show组件中的created()钩子中调用bus监听这个事件，并接收参数：
```
created() {  
  Bus.$on('getTarget', target => {  
    console.log(target);  
  });  
}  
```
这样，在每次click组件的点击事件中，就会把event.target传递到show中，并console出来。

所以eventBus的使用还是非常便捷的，但是如果是中大型项目，通信比较复杂，还是建议大家直接使用vuex。
##### 7. 使用vuex
详情见 
https://github.com/tiankongyujiao/vue/blob/master/vuex%E7%9A%84%E7%AE%80%E5%8D%95%E7%90%86%E8%A7%A3.md    
https://github.com/tiankongyujiao/vue/blob/master/vuex%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9.md
#### 8. $parents/$children/ref
```
// ref的使用
<child ref="child"></child>

在model中这样使用：
this.$refs.child
```
$parents访问左右的父组件，$children访问所有的子组件，知道顺序可以使用数组下标访问某个子组件，可以访问组件的data或者methods方法。
