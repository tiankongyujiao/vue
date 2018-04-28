#### vue如何引入jquery和jquery-ui
首先要全局引入jquery，保证引入jquery-ui时可以正常使用。
不需要任何其他的插件配合，只要将下面的代码添加到所有的loader之前

```
// 在webpack.base.conf.js中
 {
    test: require.resolve('jquery'),
    loader: 'expose?jQuery!expose?$'
 }
 
 // 例如
 module: {
    rules: [
        // 为了是jquery全局可用
        {
            test: require.resolve('jquery'),
            loader: 'expose-loader?jQuery!expose-loader?$'
        },
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: vueLoaderConfig
      },
      {
        test: /\.js$/,
        loader: 'babel-loader',
        include: [resolve('src'), resolve('test'), resolve('node_modules/webpack-dev-server/client')]
      },
      ...其他loader
   ]
  },
```
引用时改为如下方式
在引入jquery时：
```
import $ from 'expose-loader?$!jquery';
```
在当前文件或者组件中引用jquery-ui
```
import 'jquery-ui' //插件可用 
```
