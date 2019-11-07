#### vue-router的简单理解

vue可以构建一个单页面应用，既然是单页面应用，就少不了路由，vue的路由简单好用，下面介绍重点部分：

##### 1. 基本使用
创建路由，挂在到实例
```
import Router from 'vue-router'; // 首先引入vue-router

// 注意：这里使用es异步加载模块，需要使用模块才会加载进来
const foo = import('@/src/view/foo'); 
const bar = import('@/src/view/bar');
const baz = import('@/src/view/baz');
let routers = [
  { path: '/foo', component: foo },
  { path: '/bar', component: bar },
  { path: '/baz', component: baz }
]
const router = new Router({
  mode: 'history',  // vue-router 两种模式 hash和history，hash带‘#’很丑
  routers // ES6缩写语法，相当于routes:routes
})

// 最后创建vue实例并挂载
const app = new Vue({
  el: '#app',
  router
});
或者：
const app = new Vue({
  router
}).$mount('#app')
 ```
 上面是一个完整的路由实现js部分    
 html部分
 ```
 <div id="app">
    <router-view/>
 </app>
 ```
 上面`<router-view />`部分会替换成路由加载进来的页面，如果有嵌套路由，在子页面中也是同样的逻辑，则在路由实现中要使用children选项，如：
```
const routers = [
  { path: '/foo', component: foo, children:[{
    path: '/fooChild',
    component: fooChild
  }] },
  { path: '/bar', component: bar },
  { path: '/baz', component: baz }
]
```
则在foo.vue页面中使用`<router-view />`可以取到路由匹配到的fooChild。    

#### 动态路由
在路由后面加上`/:id`的形式可以实现动态路由，如：
```
const routers = [
  { path: '/foo/:id', component: foo },
  { path: '/bar', component: bar },
  { path: '/baz', component: baz }
]
```
页面访问`/foo/123`可以访问到foo组件，并且可以通过`this.$route.params.id`获取到参数id的值123    
像这种动态路由切换时，原来的组件实例会被复用，组件并不会触发生命周期函数，复用组件时，想对路由参数的变化作出响应的话，你可以简单地 watch (监测变化) $route 对象：
```
const foo = {
  template: '...',
  watch: {
    '$route' (to, from) {
      // 对路由变化作出响应...
    }
  }
}
```

#### 跳转路由
1.可以在页面中使用`<router-link to="/foo"></router-link>`的方式实现路由跳转，to表示路由要去的方向，可以是一个字符串路径，也可以是一个对象，如：
```
<router-link to="{name: 'foo', params: {id: 6}}"></router-link>
// 这是一个指向命名路由的方式
// 注意：如果是to里面的对象是path，如：
<router-link to="{path: 'foo', params: {id: 6}}"></router-link>
那么它后面使用的params将被忽略
```
2.还可以使用编程式导航实现路由跳转，如:
```
// 字符串
router.push('foo')

// 对象
router.push({ path: 'foo' })

// 命名的路由
router.push({ name: 'foo', params: { id: '123' }})

// 带查询参数，变成 /foo?id=private
router.push({ path: 'foo', query: { id: '123' }})
```

#### 导航守卫
导航解析流程    
```
1、导航被触发。    
2、在失活的组件里调用离开守卫(beforeRouteLeave)。    
3、调用全局守卫(beforeEach )。    
4、在重用的组件里调用更新守卫( beforeRouteUpdate) (2.2+)。    
5、在路由配置里调用 beforeEnter。    
6、解析异步路由组件。    
7、在被激活的组件里调用组件内守卫beforeRouteEnter。    
8、调用全局的 beforeResolve 守卫 (2.5+)。    
9、导航被确认。    
10、调用全局的 afterEach 钩子。
11、触发 DOM 更新。
12、用创建好的实例调用 beforeRouteEnter 守卫中传给 next 的回调函数
```
导航守卫有全局守卫，路由守卫，单个组件守卫：
全局守卫：    
```
const router = new Router({ ... })
// 全局前置后卫
router.beforeEach((to, from, next) => {
  // 前置守卫可以用来判断是否登录，如果没有登录，则跳转到登录页面
  // next方法一定要调用，否则钩子就不会被 resolved
  // 注意：路由匹配到的所有路由记录会暴露为 $route 对象 (还有在导航守卫中的路由对象) 的 $route.matched 数组。因此，我们需要遍历 $route.matched 来检查路由记录中的 meta 字段 `{path: '/foo', component: foo, meta: { requireAuth: true, }}`
  if (to.matched.some(record => record.meta.requireAuth)) {
      if (sessionStorage.isLogin == undefined || !eval(sessionStorage.isLogin.toLowerCase())) {
          next({
              path: '/login'
          });
      } else {
          next();
      }
  } else {
      next(); // 确保一定要调用 next()
  }
})

// 全局后置守卫
router.afterEach((to, from) => {
  document.title = to.name; // 后置守卫可以用来修改页面的title
})
```
路由独享守卫：
```
const router = new Router({
  routes: [
    {
      path: '/foo',
      component: Foo,
      beforeEnter: (to, from, next) => {
        // ...
      }
    }
  ]
})
```
组件内守卫：`beforeRouteEnter`, `beforeRouteUpdate`, `beforeRouteLeave`      
`beforeRouteEnter`: 里面调用不到this，因为守卫在导航确认前被调用,因此即将登场的新组件还没被创建。可以通过传一个回调给 next来访问组件实例。在导航被确认的时候执行回调，并且把组件实例作为回调方法的参数：
```
beforeRouteEnter (to, from, next) {
  next(vm => {
    // 通过 `vm` 访问组件实例
  })
}
```
 `beforeRouteUpdate`：离开守卫通常用来禁止用户在还未保存修改前突然离开。该导航可以通过 next(false) 来取消
```
beforeRouteLeave (to, from , next) {
  const answer = window.confirm('Do you really want to leave? you have unsaved changes!')
  if (answer) {
    next()
  } else {
    next(false)
  }
}
```


