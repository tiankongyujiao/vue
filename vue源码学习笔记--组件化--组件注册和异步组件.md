### vue源码学习笔记--组件化--组件注册和异步组件
## 一.组件注册
组件有内置组件，例如keep-alive,transition,transition-group，我们还可以自定义自己的通用组件或者业务组件，通用组件一般采用**全局注册**：
```
Vue.component('my-component', {
  // 选项
})
```
业务组件一般采用**局部注册**，在单文件组件中直接导入另一个单文件组件，然后在compontents属性上注册引入进来的单文件组件：
```
import HelloWorld from './components/HelloWorld'

export default {
  components: {
    HelloWorld
  }
}
```
### 1.全局注册
Vue.components的定义位置是在‘src\core\global-api\assets.js’中定义，在‘src\core\global-api\index.js’中调用，看下assets.js中的定义如下：
```

import { ASSET_TYPES } from 'shared/constants'
import { isPlainObject, validateComponentName } from '../util/index'

export function initAssetRegisters (Vue: GlobalAPI) {
  /**
   * Create asset registration methods.
   */
  ASSET_TYPES.forEach(type => {
    Vue[type] = function (
      id: string,
      definition: Function | Object
    ): Function | Object | void {
      if (!definition) {
        return this.options[type + 's'][id]
      } else {
        /* istanbul ignore if */
        if (process.env.NODE_ENV !== 'production' && type === 'component') {
          validateComponentName(id)
        }
        if (type === 'component' && isPlainObject(definition)) {
          definition.name = definition.name || id
          definition = this.options._base.extend(definition)
        }
        if (type === 'directive' && typeof definition === 'function') {
          definition = { bind: definition, update: definition }
        }
        this.options[type + 's'][id] = definition
        return definition
      }
    }
  })
}
```
ASSET_TYPES的定义是在'shared/constants'文件中：
```
export const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]
```
其中之一就是component组件
