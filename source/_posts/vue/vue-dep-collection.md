---
title: vue响应式原理
date: 2018-07-04 00:26:15
---

在上几篇文章介绍过，vue实现响应式是基于Object.defineProperty，但是响应式是一个复杂的过程仅仅Object.defineProperty是不行的，这一篇文章让我们揭开它神秘的面纱
<!-- more -->

### 1. 数据初始化

数据初始化的过程在以前都已经简单交代过，这次我们不再赘述直接说一点核心的东西，找到Observer类

```
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that has this object as root $data

  constructor (value: any) {
    this.value = value
    // Dep类提供了一些函数对所有的watcher进行增删改查
    // Dep还提供了一个taget字段，在依赖收集阶段至关重要
    this.dep = new Dep()
    this.vmCount = 0
    // 给当前对象新增__ob__对象，该对象指向Observer类
    def(value, '__ob__', this)
    // 如果当前数据类型为数组走此流程
    if (Array.isArray(value)) {
      const augment = hasProto
        ? protoAugment
        : copyAugment
      augment(value, arrayMethods, arrayKeys)
      this.observeArray(value)
    } else {
      // 如果当前数据类型为非数组走此流程
      this.walk(value)
    }
  }
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
    // 当前数据用defineProperty包装一下
      defineReactive(obj, keys[i])
    }
  }
  ...
}

```

defineReactive函数是给数据所有字段加上了get和set方法

```
/**
 * Define a reactive property on an Object.
 */
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
// new dep实例，相当于每个data的数据自己单独维护一个Dep
  const dep = new Dep()
 ...
  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }
  // 此处为重点，如果当前val为object或者array，那么会继续递归走上面observe -> new Observer()的过程，否则什么效果也没有，继续进行下面的流程
  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
    // 返回当前字段对应的value
      const value = getter ? getter.call(obj) : val
      // 此处是收集依赖的地方，当Dep.targe存在的时候，会调用dep.depend()往dep对象内的subs数组内push，当然push之前还会判断是否重复
      if (Dep.target) {
        dep.depend()
        // 当前value的含有子object或者array，会同时把当前的watcherpush到子元素的subs队列内
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      // 调用notify方法，循环执行subs里面的所有watcher对象内的expression回调函数
      dep.notify()
    }
  })
}
```
数据初始化是第一步，下面让我们开始第二部，模版渲染过程

###  2. 模版渲染
上面数据初始化阶段发生在create之前、beforeCreate之后，当数据初始化完以后就会就会进入渲染模版的流程

```
// 模版渲染阶段会调用该函数
function mountComponent() {
...
  updateComponent = function () {
    vm._update(vm._render(), hydrating);
  };
  new Watcher(vm, updateComponent, noop, null, true /* isRenderWatcher */);
}
```


模版渲染过程大致是，调用_render方法，然后解析html标签，生成一个新的render函数，如下

```
const render = function anonymous() {
  with(this){return _c('div',{attrs:{"id":"app"}},[_v("\n"+_s(link)+"\n"+_s(abc2)+"\n")])}
}
vnode = render.call(vm._renderProxy, vm.$createElement);
```

拿到这个函数以后把当前组件的实例传入进去，然后通过with的特性，他会自动获取link的值，最终导致触发该字段的get函数，咱们回看一下上面的get函数，他有一个判断就是Dep.target存在才会进入依赖收集过程，那么Dep.targe是怎么赋值的呢？继续往下看

### 模版渲染之前发生了什么？

上面模版渲染之前其实还有一个重要的流程没有讲到，现在我们梳理一下流程，当数据初始化完成以后，会调用mounte函数，这个函数实现如下(简化了代码，只保留了重要的部分)

```
function mountComponent() {
   updateComponent = function () {
      vm._update(vm._render(), hydrating);
    };
    new Watcher(vm, updateComponent, noop, null, true /* isRenderWatcher */);
}
```

在调用渲染模版函数的时候会new Watcher，把updateComponent复制给[watcher](https://github.com/vuejs/vue/blob/dev/src/core/observer/watcher.js)内的getter函数，当我们改完值以后触发对应回调函数就是触发的这个getter函数

```
export default class Watcher {
  vm: Component;
  expression: string;
  cb: Function;
  id: number;
  deep: boolean;
  user: boolean;
  computed: boolean;
  ...

  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
  // 判断当前是否是动态计算属性
    if (this.computed) {
      this.value = undefined
      this.dep = new Dep()
    } else {
      // 如果不是胴体啊计算属性直接调用this.get方法
      this.value = this.get()
    }
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get () {
  // 把当前的watcher对象复制给Dep.target
    pushTarget(this)
    let value
    const vm = this.vm
    try {
    // 执行你传入的回调函数，此处应该是updateComponent
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        traverse(value)
      }
      // 最后把Dep.target置为undefined
      popTarget()
      this.cleanupDeps()
    }
    return value
  }

}
```
上述代码是在我们执行_render方法之前先new watcher，在watcher内把当前的this赋值给Dep.target，然后执行updateComponent，其实本质上是执行this.update(this._render())方法，接着渲染模版的时候会从data内取值，然后触发该字段对应的get方法，这个时候Dep.target已经有值了，就会进入watcher收集的方法内，把当前的watcher放到该字段的dep对象内的subs数组内

***watcher代表了这个数据改变以后你要执行什么回调函数***
### watcher的防重机制

上文中只是简单描述了下收集过程，他其实还有防止重复机制，比如一个data字段，在模版内好几个地方都有引用，那么就会多次触发get方法造成wacher重复收集，我们看一下vue怎么处理的[watcher.js](https://github.com/vuejs/vue/blob/dev/src/core/observer/watcher.js)

```
// 新增watcher的时候会判断当前depids内是否已经包含当前watcher的id，如果不包含才可以添加到dep的subs数组内
addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }
```

### 动态计算属性

上面我们说到了data内的数据收集过程，那么动态计算属性收集过程是什么样子的呢？看下面的例子

```
<template>
  {{link2}}
  
<\/template>
<script>
 export default {
   data () {
     return {
       link: 'https://www.sinker.club'
     }
   },
   computed () {
     link2: {
       return this.link
     }
   }
 }
</script>
```
上面代码比较简单，动态计算属性link2取的是data内的link字段，这个时候是怎么收集的呢？

想搞明白这个问题就需要回头看一下动态计算属性是怎么初始化的[initComputed](https://github.com/vuejs/vue/blob/dev/src/core/instance/state.js)

```
function initComputed (vm: Component, computed: Object) {
  // 把当前所有的watcher挂载到vm._computedWatchers对象上
  const watchers = vm._computedWatchers = Object.create(null)
  // computed properties are just getters during SSR
  const isSSR = isServerRendering()

  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(
        `Getter is missing for computed property "${key}".`,
        vm
      )
    }

    if (!isSSR) {
      // 每一个动态计算属性都要单独new watcher，然后存储到vm._computedWatchers内
      watchers[key] = new Watcher(
        vm,
        getter || noop, // 本质上是动态计算属性的函数表达式
        noop,
        computedWatcherOptions
      )
    }

    // 动态计算属性初始化get和set方法
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    } else if (process.env.NODE_ENV !== 'production') {
      if (key in vm.$data) {
        warn(`The computed property "${key}" is already defined in data.`, vm)
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(`The computed property "${key}" is already defined as a prop.`, vm)
      }
    }
  }
}
```

每一个动态计算属性都初始化了一个watcher，并且回调函数是当前动态计算属性的表达式，然后以计算属性为key，watcher为value存储到_computedWatchers对象内

接下来我们看一个动态计算属性的get方法是怎么实现的

```
function createComputedGetter (key) {
  return function computedGetter () {
  // 把当前key对应的watcher取出来
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) {
      // 此处代码本质上是把当前的watcher对象赋值给Dep.target
        watcher.evaluate()
      }
      // 进入依赖收集
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```

那么现在我们基于上述demo代码扩展一下

```
<template>
 {{linke2}}
 {{linke}}
<\/template>
<script>
 // ...省略
</script>
```
这个例子我们可以看到模版内即引用的data的link也引用了动态计算属性link2，不知道大家会不会想到一个问题，当模版编译前会new watcher，回调函数对应的是updateComponent，当前dep.target指向当前的watcher，但是当获取动态计算属性值的时候，会触发this.link的get方法，那么现在Dep.target是指向的动态计算属性对应的watcher，这个是没问题的，但是当动态计算属性完成以后继续获取模版内link的时候，当前Dep.target指向哪个watcher？

要明白这个问题就需要看一下[pushTarget](https://github.com/vuejs/vue/blob/dev/src/core/observer/dep.js)、[popTarget](https://github.com/vuejs/vue/blob/dev/src/core/observer/dep.js)两个函数

```
export function pushTarget (_target: ?Watcher) {
  if (Dep.target) targetStack.push(Dep.target)
  Dep.target = _target
}

export function popTarget () {
  Dep.target = targetStack.pop()
}
```
简单文字描述一下上面会发生什么，初始化的时候Dep.targe指向初始化的updateComponent这个watcher，当我编译到动态计算属性的时候，Dep.target指向computed这个watcher并且会把以前的watcher 保存到targetStack这个数组内，当动态计算属性执行完收集依赖这个过程以后会触发popTarget函数，然后把当前的dep.target又指向回了初始化的updateComponent这个watcher

#### 响应式-->触发watcher
这一块就比较简单了，不过数组和普通的数据类型也是有点不一样的，当数组改变的时候，他会调用当前数组对象上的ob.dep.notify()
而对象的话，会从当前作用域内找到dep对象然后执行notify()，只是取值的方式不一样，本质上还是一样的。

##### 当我改变完数据以后会立马更新视图吗？
数据改变以后不会立马更新视图，他会把所有需要执行的watcher放到一个队列内，然后放到事件队列最后面循环执行，一般都是通过vue $nextTick获取数据改变以后的dom节点，他的实现原理是把你的回调函数通过promise.then/MozMutationObserver/setTimeout放到事件队列最后面

### 总结

1. 个人认为依赖收集是vue框架相对复杂的部分，我也只是尝试从自己的角度去解读，可能还有很多疏漏的地方，望多多指教
2. 依赖收集只有把这一块搞清楚你才能真的明白响应式的原理，从而更加从容的面对框架底层的一些问题


