### vue源码学习笔记--组件化--组件注册
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
          definition = this.options._base.extend(definition) // this.options._base在src\core\global-api\index.js赋值： Vue.options._base = Vue
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
在这里初始化了三个全局函数，其中之一就是component组件，如果type是component切definition是一个对象的话，就调用Vue.extend把definition对象转换成一个继承Vue的构造函数，然后通过**this.options[type + 's'][id] = definition**把构造函数挂载在this.options.components上面。  
每个组件的创建都是通过Vue.extend而来。  
在创建vnode的过程中会调用_createElement方法：
```
export function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  
  // ...
  
  let vnode, ns
  if (typeof tag === 'string') {
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if (config.isReservedTag(tag)) {
      // platform built-in elements
      if (process.env.NODE_ENV !== 'production' && isDef(data) && isDef(data.nativeOn)) {
        warn(
          `The .native modifier for v-on is only valid on components but it was used on <${tag}>.`,
          context
        )
      }
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      // unknown or unlisted namespaced elements
      // check at runtime because it may get assigned a namespace when its
      // parent normalizes children
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
  }
  
  // ...
  
}
```
在这里会走到else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {逻辑，通过resoveAsset拿到该组件的构造函数，然后执行createComponent创建组件vnode。看下resoveAsset方法的实现，在‘src\core\util\options.js’文件最后：
```
export function resolveAsset (
  options: Object,
  type: string,
  id: string,
  warnMissing?: boolean
): any {
  /* istanbul ignore if */
  if (typeof id !== 'string') {
    return
  }
  const assets = options[type]
  // check local registration variations first
  if (hasOwn(assets, id)) return assets[id]
  const camelizedId = camelize(id)
  if (hasOwn(assets, camelizedId)) return assets[camelizedId]
  const PascalCaseId = capitalize(camelizedId)
  if (hasOwn(assets, PascalCaseId)) return assets[PascalCaseId]
  // fallback to prototype chain
  const res = assets[id] || assets[camelizedId] || assets[PascalCaseId]
  if (process.env.NODE_ENV !== 'production' && warnMissing && !res) {
    warn(
      'Failed to resolve ' + type.slice(0, -1) + ': ' + id,
      options
    )
  }
  return res
}
```
>这段逻辑很简单，先通过 const assets = options[type] 拿到 assets，然后再尝试拿 assets[id]，这里有个顺序，先直接使用 id 拿，如果不存在，则把 id 变成驼峰的形式再拿，如果仍然不存在则在驼峰的基础上把首字母再变成大写的形式再拿，如果仍然拿不到则报错。这样说明了我们在使用 Vue.component(id, definition) 全局注册组件的时候，id 可以是连字符、驼峰或首字母大写的形式。  
那么回到我们的调用 resolveAsset(context.$options, 'components', tag)，即拿 vm.$options.components[tag]，这样我们就可以在 resolveAsset 的时候拿到这个组件的构造函数，并作为 createComponent 的钩子的参数。  

其中全局注册的的组件通过resoveAssert是在原型上拿的方法，而局部注册的组件自身的components就有相应的组件，因为合并组件的策略如下：
```
function mergeAssets (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): Object {
  const res = Object.create(parentVal || null)
  if (childVal) {
    process.env.NODE_ENV !== 'production' && assertObjectType(key, childVal, vm)
    return extend(res, childVal)
  } else {
    return res
  }
}
```
首先把父亲通过Object.create方法挂到res的原型上，然后如果有子元素，再在res上面合并属性，如果没有子元素，则直接返回res，所以全局注册的组件在原型上，而局部注册的组建在自身组件的components上。

### 2.局部注册
在组件实例化阶段有个合并options的逻辑，其中也会合并components，即把 components 合并到 vm.$options.components 上，这样我们就可以在上面resoveAsset的时候拿到这个组件的构造函数，并作为 createComponent 的钩子的参数。

