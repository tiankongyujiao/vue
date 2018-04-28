#### 1. beforeRouteEnter 和 beforeRouteUpdate 的区别
beforeRouteEnter 只页面刚开始进入时调用的生命周期钩子函数，只有页面进入时调用，如果页面有
```
// 假设当前在login页面
<router-link :to="{path: '/login', query: { a: 'zj' }}">link2</router-link>
```
点击这个链接，不会调用beforeRouteEnter，会调用beforeRouteUpdate，beforeRouteUpdate在页面刚进入时不会调用，只有通过上面这种参数改变时才会调用。    
在使用beforeRouteEnter 时，要注意加上 next(不可省略，使用方法如下)：
```
beforeRouteEnter (to, from, next) {
    // react to route changes...
    // don't forget to call next()
    next(vm => {  // 不可省略
      // 通过 `vm` 访问组件实例
      console.log(to,from,'enter')
      console.log(this); // undefined
      console.log(vm); // 打印出vm实例
    }) 
}
```
在使用beforeRouteUpdate 时，同样要加next(), 和beforeRouteEnter 有所不同：
```
beforeRouteUpdate (to, from, next) {
  // react to route changes...
  // don't forget to call next()
  console.log(to,from,'test');
  console.log(this); // 打印出vm实例
  next(); // 不可省略
}
```
注意： 在beforeRouteEnter 里 还没有this变量（undefined），所以此时带入了参数vm；在 beforeRouteUpdate 里可以直接使用this。
#### 2.动态路由和嵌套路由
个人理解，动态路由就是在url里面写入参数，页面可以获取到这个参数，这个是通过‘/’隔开的参数，和url里面‘?’的query有所不同。    
url虽然不一样，但是导入的组件是一样的，这时候就用到了动态路由。
```
{
    path: '/register/:id',
    name: 'register',
    component: register,
    children: [
        {
          // 当 /user/:id/profile 匹配成功，
          // UserProfile 会被渲染在 User 的 <router-view> 中
          path: 'profile',
          component: profile
        },
        {
          // 当 /user/:id/posts 匹配成功
          // UserPosts 会被渲染在 User 的 <router-view> 中
          path: 'posts',
          component: posts
        }
    ]
}
```
上面例子所示，通过 http://localhost:8081/register/1 或者 http://localhost:8081/register/2 都能访问到register组件（1和2就是对应的id），此时可以在页面通过this.$route.params得到‘1’或者‘2’。

嵌套路由：不仅在app.vue这样的根实例文件中可以使用<router-view />渲染页面，在组件中同样可以使用<router-view />嵌入子组件，如上示例中也是一个嵌套路由的例子，组件的嵌套组件路由写在组件的children中，注意，此时如果组件渲染‘profile’子组件，url链接地址应该是这样的：http://localhost:8081/register/2/profile （如果子组件定义了动态路由，即加了参数，要一直带着参数，访问组件嵌套组件，直接跟着'/子组件path'即可）。 

如果目的地和当前路由相同，只有参数发生了改变 (比如从一个用户资料到另一个 /register/1 -> /register/2)，你需要使用 beforeRouteUpdate 来响应这个变化 (比如抓取用户信息)

#### 3.向路由组件传递props
上面说了动态路由在页面中可以通过this.$route.params访问，在组件中使用 $route 会使之与其对应路由形成高度耦合，从而使组件只能在某些特定的 URL 上使用，限制了其灵活性。可以使用 props 将组件和路由解耦。
```
const User = {
  props: ['id'],
  template: '<div>User {{ id }}</div>'
}
const router = new VueRouter({
  routes: [
    { 
        path: '/user/:id', 
        component: User, 
        props: true 
    },

    // 对于包含命名视图的路由，你必须分别为每个命名视图添加 `props` 选项：
    {
        path: '/user/:id',
        components: { default: User, sidebar: Sidebar },
        props: { default: true, sidebar: false }
    }
  ]
})
```
在组件中使用props，在路由中加参数props：true。

如果 props 被设置为 true，route.params 将会被设置为组件属性。

如果 props 是一个对象，它会被按原样设置为组件属性。
```
const router = new VueRouter({
  routes: [
    { 
        path: '/promotion/from-newsletter', 
        component: Promotion, 
        props: { newsletterPopup: false } // newsletterPopup会被设置为该组件的属性，在组件中可以访问到newsletterPopup
    }
  ]
})
```
#### 4.路由跳转的时候使用path和name都可以
```
<router-link :to="..."> // 声明式
router.push(...)  // 编程式

// 命名的路由
router.push({ name: 'user', params: { userId: 123 }})

// 带查询参数，变成 /register?plan=private
router.push({ path: 'register', query: { plan: 'private' }})
// 如果提供了 path，params 会被忽略，上述例子中的 query 并不属于这种情况
```
声明式导航，当需要使用动态路由的时候要用name，因为动态路由的路径是不固定的。同样，编程式导航如果提供path，params会被忽略，即动态路由路径跳转要使用name，而不是path。
