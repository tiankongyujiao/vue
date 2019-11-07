#### axios的简单理解
axios的底层实现是基于原生的XMLHttpRequest实现的，原生的XMLHttprequest实现方法如下：
```
const xml = new XMLHttpRequest();
xml.open('get', url, true); // 第三个参数为false表示同步请求，true表示异步请求
xml.send();
xml.onReadyStateChange = function(){
  if(sml.readyState === 4 && xml.status === 200){
    // 在这里把请求返回的结果xml.responseText赋值给需要用到的地方
  }
}
```
而axios是使用promise对XMLHttpRequest进行了一层封装：
```
function ajax(url){
  return new Promise(function(resolve, reject){
    const xml = new XMLHttpRequest();
    xml.open('get', url, true); // 第三个参数为false表示同步请求，true表示异步请求
    xml.send();
    xml.onReadyStateChange = function(){
      if(sml.readyState === 4 && xml.status === 200){
        resolve({data: xml.responseText}); // axios封装的时候是data属性名，所以在使用axios取数据时，resp.data才是后台返回的值
      }
    }
  })
}

ajax(url).then(function(resp){
  // 在这里可以通过resp.data可以拿到data的值
})
```
#### axios的基本使用
1.可以使用`axios.get({})`和`axios.post({})`方式，如：
```
axios.get('url',{
  params:{ // get的方式一般使用params，post方式使用data
    id:'接口配置参数（相当于url?id=xxxx）'，
  },
}).then(function(res){
  console.log(res);//处理成功的函数 相当于success
}).catch(function(error){
  console.log(error)//错误处理 相当于error
})

axios.post('url',
{ data:xxx },{
  headers:xxxx,
}).then(function(res){
  console.log(res);//处理成功的函数 相当于success
}).catch(function(error){
  console.log(error)//错误处理 相当于error
})
```
还可以直接使用axios API 通过相关配置传递给axios完成请求：
```
axios({
  method: 'get',
  url:'xxx',
  cache:false,
  params:{id:123},
  headers:xxx,
})
//------------------------------------------//
axios({
  method: 'post',
  url: ’/user/12345’,
  data: {
    firstName: ’Fred’,
    lastName: ’Flintstone’
  }
});
```
axios还可以设置拦截器，请求前拦截和请求后拦截：
```
const service = axios.create({
  baseURL: baseURL, 
  timeout: 6000 // 请求超时时间
})

// 请求前拦截：请求前校验是否携带了正确的token
service.interceptors.request.use(config => {
  // 需要单独安装（vue-ls: Vue plugin for work with Local Storage from Vue context）
  // vue-ls是vue用于存储localstorage的组件库
  const token = Vue.ls.get(ACCESS_TOKEN) 
  if (token) {
    config.headers['token'] = token // 让每个请求携带自定义 token 请根据实际情况自行修改
  }
  return config
}, err)

// 请求后拦截
service.interceptors.response.use((response) => {
  return response.data
}, err)
```

可以更改axios的默认属性，如：
```
axios.defaults.headers.common['token'] = sessionStorage.token; // 设置token
axios.defaults.headers.common['Accept-Language'] = localStorage.baseLang; // 设置语言
axios.defaults.headers.post['Content-Type'] = 'application/x-www-form-urlencoded'; // 设置发送文本类型
axios.defaults.withCredentials = true; //axios 默认不发送cookie
axios.defaults.baseURL= baseURL; // 设置基础url
axios.defaults.timeout = 6000; // 设置超时时间，这里为6s
```
