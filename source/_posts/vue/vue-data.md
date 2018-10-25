---
title: vue data初始化的两种方式
date: 2018-06-28 19:32:15
---

vue data有两种初始化的方式，function和object，但是这两种情况适用场景有哪些？能不能通用？带着这两个问题咱们一起分析下

<!-- more -->

#### data初始化

```
// 代码来源于官网示例

// 第一种定义方式
var data = { a: 1 }

// 直接创建一个实例
var vm = new Vue({
  data: data
})

// Vue.extend() 中 data 必须是函数
var Component = Vue.extend({
// 第二种定义方式
  data: function () {
    return { a: 1 }
  }
})
```
上述代码简单描述了data定义的两种方式
1. function
2. object

官网demo中也着重说了extend中data初始化不能用object。那么为什么呢？

#### 源码分析

按照官网demo,Vue.extend中的data初始化不能是Object，如果我们强制写成Object会出现什么？

```
var Component = Vue.extend({
  data: { a: 1 }
})
```
运行以后chrome的consolo直接报错，信息如下

```
vue.esm.js?efeb:591 [Vue warn]: The "data" option should be a function that returns a per-instance value in component definitions.
```
通过分析源码以及报错信息，当触发Vue.extend的时候，他会做一个合并操作，把一个基础组件（里面vmode, transtion等）和你定义在extend内的信息，通过mergeField往options上合并，当合并到data的时候，他会触发strats.data，在这个里面会check data是不是一个function，这里需要注意的是filter、components等和data走的是两套合并流程，详细的请看代码注释，如下

```
// vue.extend 源码地址https://github.com/vuejs/vue/blob/dev/src/core/global-api/extend.js

  Vue.extend = function (extendOptions: Object): Function {
  ...
  // 在这里会触发mergeOptions方法
  Sub.options = mergeOptions(
      Super.options,
      extendOptions
    )
  ...
}

// mergeOptions 源码地址https://github.com/vuejs/vue/blob/dev/src/core/util/options.js

export function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  ...

  const options = {}
  let key
  // parent对象内包含components、filter,、directive
  for (key in parent) {
    mergeField(key)
  }
  // child对象内对应的是Vue.extend内定义的参数
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  function mergeField (key) {
  // 这一步是根据传入的key找到不同的合并策略filter、components、directives用到合并策略是这个方法mergeAssets和data用到的不一样，当合并到data的时候会进入专属的合并策略方法内
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
}

// strats.data  源码地址https://github.com/vuejs/vue/blob/dev/src/core/util/options.js
strats.data = function (
  parentVal,
  childVal,
  vm
) {
  if (!vm) {
  // 如果data不是function的话会直接走下面的报错信息
    if (childVal && typeof childVal !== 'function') {
      process.env.NODE_ENV !== 'production' && warn(
        'The "data" option should be a function ' +
        'that returns a per-instance value in component ' +
        'definitions.',
        vm
      );

      return parentVal
    }
    return mergeDataOrFn(parentVal, childVal)
  }

  return mergeDataOrFn(parentVal, childVal, vm)
};

```

#### 其他情况

其实我们上述代码只是一个简单的流程，在实际开发中同类情况有：子组件内、路由内都不可以把data定义为一个对象，因为他们底层都调用了mergeOptions方法

#### 什么时候可以定义成一个对象

在vue初始化的时候，如下

```
new Vue({
  data: {
    linke: '//sinker.club'
  }
})
```

#### 意义

ok，上面说了那么多，那么这么做的意义是什么？为什么那几种情况不可以定义为对象？
其实回答这个问题，需要回到js本身，众所周知js数据类型分为引用和基本，引用类型包含Object, Array, Function，何为引用类型就不在这里阐述了

```
  var obj = {link: '//www.sinker.club'}
  var obj2 = obj
  var obj3 = obj
  obj2.link = "//gitlab.sinker.club"
  console.log(obj3.link) // "//gitlab.sinker.club"
```

上述代码反应了一个问题，由于obj3和obj2在内存中都是指向一个地址，那么obj2的修改会影响到obj3，当然处理这种问题可以用深copy来做到
1. JSON.parse(JSON.stringify(obj))
2. deepClone(obj)

但是这两种做法需要开发或者框架每一次都要深copy一次，当数据量大的时候对性能什么都不友好，那么Vue怎么做的呢？把data定义成一个function

```
function data() {
  return {
   link: '//sinker.club'
  }
}

var obj = test()
var obj2 = test()

obj2.link ="//gitlab.sinker.club"
console.log(obj.link) '//sinker.club'

```


#### 为什么这么做？解决的场景是什么呢？

比如我定一个子组件，data是按照对象的方式定义的，这个组件在多个地方引用，如果其中一个引用此组件的data修改了，那么就会造成其余引用此组件的data同时改变， end.

#### 吐槽

Vue源码是好几个月以前看的了，上次看的时候就是大致过了一下流程，但是里面好多细节都没有注意到，以后会不定期继续更新对Vue源码的一些心得，也欢迎各位同学拍砖吐槽，多交流



