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
