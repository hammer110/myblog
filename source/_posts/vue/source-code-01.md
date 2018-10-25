---
title: vue源码系列01
date: 2018-05-23 17:15:15
---

本文是vue源码系列的第一篇文章，会简单的介绍一下vue的初始化

<!-- more -->
#### import Vue from 'vue' 发生了什么

##### 1.&nbsp;首先打开vue/src/core.index.js

<pre>
// 初始化vue
import Vue from './instance/index'
import { initGlobalAPI } from './global-api/index'
import { isServerRendering } from 'core/util/env'
// 往config上挂载方法，然后往Vue.until上挂载一些公共方法
/*
*   warn,
*   extend,           
*   mergeOptions,    // 合并属性
*   defineReactive  // vue-router就是用的这个方法给
*   _route用defineProperty包装了一层
*/
initGlobalAPI(Vue)

// 往vue原型对象上挂载$isServer和$ssrContext两个属性
Object.defineProperty(Vue.prototype, '$isServer', {
  get: isServerRendering
})

Object.defineProperty(Vue.prototype, '$ssrContext', {
  get () {
    /* istanbul ignore next */
    return this.$vnode.ssrContext
  }
})
// 给Vue挂载静态属性 __VERSION__
Vue.version = '__VERSION__'
export default Vue
</pre>

##### 2.&nbsp; **instance/index** 初始化vue

<pre>
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'
// Vue构造函数
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  // 初始化的时候调用_init方法，这个方法是在initMixin
  // 方法内挂载Vue原型上的
  this._init(options)
}
// 只做了一件事，往Vue原型上挂载_init方法, 在钩子函数内调用
initMixin(Vue)
// 往原型对象上挂载$data, $props(用defineProperty包装了
// 一层，其实获取的是this._data/this._props), $set, 
// $delete, $watch方法
stateMixin(Vue)

/*
* 往原形对象上挂载一些组件间通信的方法
* $on, $once, $off, $emit
*/
eventsMixin(Vue)

/*
* 往原型上挂载
* _update 更新组件(内部使用)
* $forceUpdate 强制更新
* $destroy 销毁组件方法
*/
lifecycleMixin(Vue)

/*
* 往原型上挂载$nextTick, _render方法
* 在这个js底部往原型上挂载了一些方法的简称
* 这些是在编译模版的时候使用的
*/
renderMixin(Vue)

export default Vue

</pre>

##### 3.&nbsp; 其他
在初始化的时候还往原型上挂载了好多方法，这里就不一一说了有兴趣的时候可以自己看一下，后期我们会找几个重点的api，简单讲一下他的实现方式

#### new Vue({}) 发生了什么

我们写一个最简单的实例代码

<pre>
  new Vue({
    data () {
      return {}
    },
    created () {},
    mounted () {},
    computed () {}
  })
</pre>

##### 1. this._init

调用_init方法，然后把传进来的option直接传给_init, 这个方法就是页面初始化的整个过程(从初始化data到dom渲染完成)。且听我们慢慢道来

##### 2. 初始化/合并一些参数

<pre>
// 当我们每次new一个vue实例的时候，这个uid就会自动加一，
// 我的理解是类似于该实例化对象的唯一标志
 vm._uid = uid++
 // 2.2.0新增Performance
let startTag, endTag
/* istanbul ignore if */
if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
 startTag = `vue-perf-init:${vm._uid}`
 endTag = `vue-perf-end:${vm._uid}`
 mark(startTag)
}

// 合并属性到$options上
if (options && options._isComponent) {
 initInternalComponent(vm, options)
} else {
 vm.$options = mergeOptions(
   resolveConstructorOptions(vm.constructor),
   options || {},
   vm
 )
}

if (process.env.NODE_ENV !== 'production') {
 initProxy(vm)
} else {
 vm._renderProxy = vm
}

// expose real self
vm._self = vm
// 往vm上挂载一些参数
initLifecycle(vm)

// 目前只能分析出来是往vm上挂载_events对象，但是下面
// 还有一部分代码看不太明白
initEvents(vm)

// 往vm上挂载 _c(在compile模版阶段使用)和$createElement两个方法
initRender(vm)

// 上面已经把你实例化传进来的配置参数合并到vm.$options上
// 然后从vm.$options上取出来beforeCreate然后执行
callHook(vm, 'beforeCreate')

// https://cn.vuejs.org/v2/api/#provide-inject
initInjections(vm)

/*
* 初始化数据，如果组件内有props包装一层，然后往原型上挂载
* _props，methods直接挂载vm实例上（这个也就是你为什么可以直* 接用vm.methodName直接调用）
* 最后initData，初始化你配置在data里面的数据, 并且把data内的数据直接挂载到vm上，用proxy
* 方法，后期编译模版把data的值赋值到vnode中也是借助于此处处理的结果
*/
initState(vm)

// https://cn.vuejs.org/v2/api/#provide-inject
initProvide(vm)
// 执行你配置在created里面的函数
callHook(vm, 'created')
 /* istanbul ignore if */
if (process.env.NODE_ENV !== 'production' && s && mark) {
 vm._name = formatComponentName(vm, false)
 mark(endTag)
 measure(`${vm._name} init`, startTag, endTag)
}
// 初始化页面vm.$mount
if (vm.$options.el) {
 vm.$mount(vm.$options.el)
}
}
</pre>

### 总结

今天只是简单介绍了一下vue的初始化流程，我们后续会逐步把这些内容展开。vm.$mount他是怎么编译模版的，更新data的数据的时候ui是怎么更新的等等，敬请期待

