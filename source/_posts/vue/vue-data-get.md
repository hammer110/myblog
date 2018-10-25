---
title: vue获取data值的方式分析
date: 2018-06-29 06:10:15
---

上一篇文章我们简单讲解了data初始化的两种方式，这次我们分析一下获取data内值的方式

<!-- more -->

#### 获取vue的data
我们常用获取data值的方式为如下两种：
1. this.$data.link
2. this.link

不知道大家有没有一个疑问，按照我们定义的component内的data，获取的方式应该是***this.data.link***，那么现在的获取Vue data值的方式是怎么实现的呢？来让我们一起分析下


#### 引入Vue的时候发生了什么？

当我们引入Vue的时候会往Vue的prototype上挂在一些方法

***import Vue from vue***的时候会加载[src/core/index.js](https://github.com/vuejs/vue/blob/dev/src/core/index.js)，在这个里面做一些初始化

```
// 这个是重点
import Vue from './instance/index'
import { initGlobalAPI } from './global-api/index'
import { isServerRendering } from 'core/util/env'
import { FunctionalRenderContext } from 'core/vdom/create-functional-component'

initGlobalAPI(Vue)
...
export default Vue
```

[instance/index.js](https://github.com/vuejs/vue/blob/dev/src/core/instance/index.js)这个文件做了一些vue的初始化的工作

```
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

...
// 这个是重点
stateMixin(Vue)
...

export default Vue
```

[stateMixin](https://github.com/vuejs/vue/blob/dev/src/core/instance/state.js) 这个方法会往vue prototype挂在$data等属性

```
export function stateMixin (Vue: Class<Component>) {
    const dataDef = {}
    // 当this.$data的时候返回的是this._data
    dataDef.get = function () { return this._data }
    const propsDef = {}
    propsDef.get = function () { return this._props }
    Object.defineProperty(Vue.prototype, '$data', dataDef)
    Object.defineProperty(Vue.prototype, '$props', propsDef)
}
```
通过上述代码可以发现，当我们this.$data的时候本质上是取的this._data

#### _data又是什么鬼？

想要搞明白这个就需要我们继续分析一下Vue的实例化过程


```
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}
```

Vue的构造函数比较简单，就是直接调用了this.init方法. [instance/init.js](https://github.com/vuejs/vue/blob/dev/src/core/instance/init.js)

```
export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    ...
    vm._self = vm
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    // 这个是重点
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')
}
```
[instance/init.js](https://github.com/vuejs/vue/blob/dev/src/core/instance/init.js)按照Vue的生命周期做了一些初始化工作，在这里只需要关注[initState方法](https://github.com/vuejs/vue/blob/dev/src/core/instance/state.js)

initState方法底层调用了initData给所有的data数据做二次包装，所以我们直接看initData的具体实现

```
function initData (vm: Component) {
// 在这一步把data赋值给了this._data
let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  ...
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
    // 这里是重点，会把data上的数据copy到_data上
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  observe(data, true /* asRootData */)
}
```
这一步在方法顶部会执行data的方法，然后把返回的值赋值给_data，其实到这一步就解决了我们第一个问题为什么***this.$data.link***能读取到data内的值，因为当我们this.$data.link的时候他会触发link的get方法，get方法会读取this._data.link。ok，上面只是解决了第一个问题，那么第二个this.link又是怎么获取到值的呢？继续往下看

通过分析发现在初始化的过程最后还会调用[proxy](https://github.com/vuejs/vue/blob/dev/src/core/instance/state.js)方法

```
export function proxy (target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  // target===vm
  // key === link
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

这个方法比较简单，定义了get和set方法，然后把所有data内的值都直接挂载到vm上，当我们this.link的时候触发get方法然后会直接return this._data.link


#### 小结

上面写的比较啰嗦，因为涉及到初始化的一些过程，但本质上还是很简单的，不管是this.$data.link还是this.link都是从this._data上获取的

#### 补充
我们上面讲了data，那么methods内的方法为什么可以直接通过this调用呢？是不是和data的过程一样？具体可以看一下[initMethods](https://github.com/vuejs/vue/blob/dev/src/core/instance/state.js)的实现




