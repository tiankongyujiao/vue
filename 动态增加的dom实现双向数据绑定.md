```
var dsViewVue = Vue.extend({
    template: '<div>' + resp + '</div>',
    i18n,
    components:{
        editor
    },
    data: function(){
        return{
            curWidget: _this.curWidget
        }
    }
});
var dsView = new dsViewVue().$mount(); // 文档之外渲染
$('#dsView').html(dsView.$el); // 使用原生DOM API把它插入到文档中
```
##### Vue.extend( options ) 方法讲解：
使用基础 Vue 构造器，创建一个“子类”。参数是一个包含组件选项的对象。    
data 选项是特例，需要注意 - 在 Vue.extend() 中它必须是函数    
```
<div id="mount-point"></div>

// 创建构造器
var Profile = Vue.extend({
  template: '<p>{{firstName}} {{lastName}} aka {{alias}}</p>',
  data: function () {
    return {
      firstName: 'Walter',
      lastName: 'White',
      alias: 'Heisenberg'
    }
  }
})
// 创建 Profile 实例，并挂载到一个元素上。
new Profile().$mount('#mount-point')
```
##### vm.$mount( [elementOrSelector] ) 方法讲解：
如果 Vue 实例在实例化时没有收到 el 选项，则它处于“未挂载”状态，没有关联的 DOM 元素。可以使用 vm.$mount() 手动地挂载一个未挂载的实例。    
如果没有提供 elementOrSelector 参数，模板将被渲染为文档之外的的元素，并且你必须使用原生 DOM API 把它插入文档中。    
```
var MyComponent = Vue.extend({
  template: '<div>Hello!</div>'
})

// 创建并挂载到 #app (会替换 #app)
new MyComponent().$mount('#app')

// 同上
new MyComponent({ el: '#app' })

// 或者，在文档之外渲染并且随后挂载
var component = new MyComponent().$mount()
document.getElementById('app').appendChild(component.$el)
```

