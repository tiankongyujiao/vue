### reactive, ref
通过reactive和ref包装一层，把字符串包装进ref，或者把对象包装进reactive中，ref也可以包装对象，但是内部还是会用reactive包装一层的，调用响应式的reactive和ref方法。  
ref调用了createRef方法，直接在createRef方法内设置了get和set函数，在get内track收集依赖，在set方法内触发更新trigger。  
reactive是使用了new Proxy，同样也是在get内track收集依赖，在set方法内触发更新trigger

### track收集依赖

### trigger触发更新

### effect副作用函数，当依赖发生变化，触发副作用函数执行
