---
title: Vue 数组更新机制
date: 2018-06-30 13:59:15
---

众所周知Vue实现响应式是基于Object.definePropertie，这个方法是Object上，那么数组是怎么实现的呢？

<!-- more -->

### 初始化

数组和对象的初始化是不一样的，Vue源码中会通过Array.isArray来区分当前变量是不是数组，然后调用不同的方法，本次只简单讲一下数组，由于篇幅有限我们直接讲核心源码，实际上整个流程是[initState](https://github.com/vuejs/vue/blob/dev/src/core/instance/init.js) => [initData](https://github.com/vuejs/vue/blob/dev/src/core/instance/state.js) => [observe](https://github.com/vuejs/vue/blob/dev/src/core/observer/index.js) => [Observer](https://github.com/vuejs/vue/blob/dev/src/core/observer/index.js)

```
export class Observer {
  constructor (value: any) {
    this.value = value
    // 此处也是重点
    this.dep = new Dep()
    this.vmCount = 0
    // 给所有的属性身上都新增一个变量__ob__
    def(value, '__ob__', this)
    // 如果判定为数组的话调用数组专有的方法
    if (Array.isArray(value)) {
      // hasProto= "__proto__" in hasProto
      const augment = hasProto
        ? protoAugment
        : copyAugment
        // 此处也是关键，这个是把所有Vue封装的数组变异方法覆盖到数组的__proto__上， 请记住是整体覆盖__proto__
      augment(value, arrayMethods, arrayKeys)
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }
}
```

数组一般分为两种json数组或者普通数据类型的数组，如下

```
var arr = [{id: 1}]
var arr = [1, 2, 3]
```
如果是第一种的话，Vue会继续递归，直到给每个对象都加上__ob__，如果是第二种的话，给在这里这个数组加上__ob__就完事了

### __ob__是什么东西？

查看上面的代码可以发现__ob__是Observer的对象的copy，重点是new Dep那里，因为dep对象是管理所有watcher的地方，他是Vue实现某个数据改变更新视图/调用对应的回调函数的关键

### arrayMethods

上面有一处代码是把所有的Vue封装的数组方法覆盖到当前数组的原型上__proto__，那么Vue具体封装了那么数组方法呢？ [Observer/array.js](https://github.com/vuejs/vue/blob/dev/src/core/observer/array.js)

```
import { def } from '../util/index'
const arrayProto = Array.prototype
export const arrayMethods = Object.create(arrayProto)

const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]

methodsToPatch.forEach(function (method) {
  // cache original method
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator (...args) {
  // 调用原始的数组方法
    const result = original.apply(this, args)
    const ob = this.__ob__
    let inserted
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    if (inserted) ob.observeArray(inserted)
    // 调用所有该值对应的watcher
    ob.dep.notify()
    return result
  })
})

```

通过分析上面代码可以看到Vue封装了[push, splice, ...]等方法，在最后会调用该值__ob__的notify方法，也就是数组响应式的本质

### 其他情况

```
<template>
   <ul><li v-for="item in arr">{{item.id}}</li></ul>
   <ul><li v-for="item in arr2">item</li></ul>
</template>
{
  data () {
    return {
      arr: [{id: 1}, {id: 2}, {id: 3}],
      arr2: [1, 2, 3]
    }
  },
  created () {
     this.arr[0] = [{id: 5}]
     this.arr2[0] = 5
     console.log(this.arr[0]) // 5
     console.log(this.arr2[0]) // 5
  }
}
```
模版里面循环arr和arr2, 当我们按照上述代码在created改变数组的值，那么按照前面的分析，此处应该只会改变数据而不会更新视图，但是实际上恰恰相反，这个是因为什么呢？我感觉是因为在created截图只是初始化了数据，而还没有渲染dom，等渲染dom的时候会从data内取值，这个时候data内的数组现在是更改完以后的

```
{
...
 mounted () {
     this.arr[0] = [{id: 5}]
     this.arr2[0] = 5
     console.log(this.arr[0]) // 5
     console.log(this.arr2[0]) // 5
  }
}
```

那么我们把更新数据的代码放到mounted会出现什么情况呢？arr改完以后视图会更新，而arr2反而不会，那么这又是什么情况呢？

arr会更新视图，我个人感觉是因为数组内的数据类型是object，因为在initData阶段会循环arr数组，如果数组内的数据是object，同样会给当前值绑定上get和set方法，所以arr直接修修改值的时候会触发对象的set方法，然后调用该值对应的watcher内的express方法(this.update)，驱动视图更新

### 总结

数组更新机制其实官网也有表述但是比较简单，只是说了Vue对数组的方法做了二次封装，但是具体怎么实现的没有写，希望此篇文章能让大家对Vue的数组有一个更深入的了解


