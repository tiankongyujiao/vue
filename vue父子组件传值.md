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
子组件通过props接收父组件的传值，通过$emit方法触发父组件事件改变传值（不能在子组件修改父组件的值），如果父组件传递给子组件的值是动态的（ajax异步请求获取来的），需要在子组件watch传值赋值给子组件自身的data属性。
