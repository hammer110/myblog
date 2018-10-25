---
title: Vue源码系列02
date: 2018-05-23 17:15:15
---

[上一篇文章](https://github.com/hammer110/vue-seed-project/wiki/vue%E6%BA%90%E7%A0%81%E7%B3%BB%E5%88%9701)简单介绍了一下Vue是如何初始化的，这次我们简单分析一下$mount以后发生的事情

#### 1. 渲染模版

上一篇我们讲到了在初始化以后，会调用vm.$mount方法
<!-- more -->
```
// 现在Vue.prototype.$mount指向的是src/web/index.js下的$mount，把她保存为mount,然后重
// 新赋值
const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el,
  hydrating
) {
  el = el && query(el);

  /* istanbul ignore if */
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== 'production' && warn(
      "Do not mount Vue to <html> or <body> - mount to normal elements instead."
    );
    return this
  }

  var options = this.$options;
  
  // 判断在组件内是否定义了render方法, 如果定义的话，会走本身的render方法
   if (!options.render) {
    var template = options.template;
    // 判断组件内是否有template字段，如果没有的话调用getOuterHTML获取指定的html字符串
    if (template) {
      if (typeof template === 'string') {
       ...
      } else if (template.nodeType) {
        template = template.innerHTML;
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this);
        }
        return this
      }
    } else if (el) {
      template = getOuterHTML(el);
    }
     }
    if (template) {
      /* istanbul ignore if */
      if ("development" !== 'production' && config.performance && mark) {
        mark('compile');
      }
      
     /**
     * 调用compile方法，对模版以及你传入自定义的delimiters进行编译, 返回一个render函数
     * function anonymous() {
     *    with(this){return _c('div',[_m(0),(message)?_c('p', *[_v(_s(message))]):_c('p',[_v("No message.")])])}
     *  }
     *
     */
     // 编译模版，返回render函数
      var ref = compileToFunctions(template, {
        shouldDecodeNewlines: shouldDecodeNewlines,
        delimiters: options.delimiters
      }, this);
      var render = ref.render;
      var staticRenderFns = ref.staticRenderFns;
      options.render = render;
      options.staticRenderFns = staticRenderFns;
     ......
    }
  }
    // 继续调用 mount方法，判断是否在浏览器环境，如果不是的话 el = undefined
  return mount.call(this, el, hydrating)
};
```

#### 2. mount会触发mountComponent(src/core/instance/lifecycle.js)函数

该函数主要做了如下几件事
1. 调用beforeMount生命周期函数
2. 定义updateComponent
3. new Watcher() 渲染页面，收集依赖
4. 调用mounted函数

#### 3. 收集依赖 ***new Watcher(vm, updateComponent, noop)***

Watcher类
> Watcher是收集依赖的关键函数

```
var Watcher = function Watcher (
  vm,
  expOrFn,
  cb,
  options
) {
  this.vm = vm;
  // 当前wather对象存储到vm上的_watchers数组内
  vm._watchers.push(this);
  // options
  // watcher可以接受4个参数，可以参考$watch的第三个参数option
  if (options) {
    this.deep = !!options.deep;
    this.user = !!options.user;
    this.lazy = !!options.lazy;
    this.sync = !!options.sync;
  } else {
    this.deep = this.user = this.lazy = this.sync = false;
  }
  this.cb = cb;
  this.id = ++uid$2; // uid for batching
  this.active = true;
  this.dirty = this.lazy; // for lazy watchers
  this.deps = [];
  this.newDeps = [];
  // 把所有依赖的dep的id都搜集起来，后面会用到，防止重复收集依赖
  this.depIds = new _Set();
  this.newDepIds = new _Set();
  this.expression = process.env.NODE_ENV !== 'production'
    ? expOrFn.toString()
    : '';
  // 把你传进来expOrFn赋值给this.getter
  if (typeof expOrFn === 'function') {
    this.getter = expOrFn;
  } else {
    this.getter = parsePath(expOrFn);
    if (!this.getter) {
      this.getter = function () {};
    }
  }
  // 由于new Watcher的时候没有传入option，所以this.lazy为false, 会触发this.get方法
  this.value = this.lazy
    ? undefined
    : this.get();
};
```

***this.get***
> this.get会触发你new Watcher的时候传进来的expOrFn，也就是this.getter函数
> 这里的getter的函数就是渲染dom结构的函数

```
get () {
   // 存储到targetStacks数组中，并且Dep.target = this
    pushTarget(this)
    let value
    const vm = this.vm
    ／/ this.user = true的时候是你在组件内定义watch/this.$watch方法的时候
    if (this.user) {
      try {
        value = this.getter.call(vm, vm)
      } catch (e) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      }
    } else {
    // 现在会触发你传进来的回调函数expOrFn，也就是updateComponent方法
      value = this.getter.call(vm, vm)
    }
    // "touch" every property so they are all tracked as
    // dependencies for deep watching
    if (this.deep) {
      traverse(value)
    }
    
    popTarget()
    this.cleanupDeps()
    return value
  }
```

#### 4. 渲染页面 updateComponent (src/core/instance/lifecycle.js)

> 该函数主要是调用_render方法，生成vnode
> 调用_update更新页面
> 如果是非生产环境并且在配置项里面配置了Performance，会mark页面渲染的用时

```
updateComponent = () => {
 const name = vm._name
 const id = vm._uid
 const startTag = `vue-perf-start:${id}`
 const endTag = `vue-perf-end:${id}`

 mark(startTag)
// 调用_render生成vnode
 const vnode = vm._render()
 mark(endTag)
 measure(`${name} render`, startTag, endTag)

 mark(startTag)
 vm._update(vnode, hydrating)
 mark(endTag)
 measure(`${name} patch`, startTag, endTag)
}
```
vm_render

```
  Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const {
      render,
      staticRenderFns,
      _parentVnode
    } = vm.$options
    let vnode
    try {
    // 调用render函数，如果页面里面获取了data里面定义的值
    // 这里会触发data的get函数，会把对应的watcher放到
    // 该data对应dep数组内
      vnode = render.call(vm._renderProxy, vm.$createElement)
    } catch (e) {
    }
  }
```

vm.__update

> vm.__patch__更新ui的核心方法，初始化的时候会根据vnode创建dom树，如果更新的时候会__patch__会比对出来，然后渲染出来最新的dom结构

```
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    // 如果代码
    if (vm._isMounted) {
      callHook(vm, 'beforeUpdate')
    }
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const prevActiveInstance = activeInstance
    activeInstance = vm
    vm._vnode = vnode
    // 初始化的时候prevVnode是null
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(
        vm.$el, vnode, hydrating, false /* removeOnly */,
        vm.$options._parentElm,
        vm.$options._refElm
      )
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    ...
  }
```
vm.__patch__调用createElm方法，创建dom
> createElm用createElement创建标签，然后children的内容或者dom，会insert进该标签，当循环到最后一层的时候，会把最外层的标签insert进页面，然后删掉原有的dom标签

```
  function createElm (vnode, insertedVnodeQueue, parentElm, refElm, nested) {
    vnode.isRootInsert = !nested // for transition enter check
    // 如果当前的标签是自定义组件，那么就会进入初始化组件方法，调用 this._init方法
    if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
      return
    }

    const data = vnode.data
    const children = vnode.children
    const tag = vnode.tag
    if (isDef(tag)) {
      if (process.env.NODE_ENV !== 'production') {
        if (data && data.pre) {
          inPre++
        }
        if (
          !inPre &&
          !vnode.ns &&
          !(config.ignoredElements.length && config.ignoredElements.indexOf(tag) > -1) &&
          config.isUnknownElement(tag)
        ) {
          warn(
            'Unknown custom element: <' + tag + '> - did you ' +
            'register the component correctly? For recursive components, ' +
            'make sure to provide the "name" option.',
            vnode.context
          )
        }
      }
      vnode.elm = vnode.ns
        ? nodeOps.createElementNS(vnode.ns, tag)
        : nodeOps.createElement(tag, vnode)
      setScope(vnode)

      /* istanbul ignore if */
      if (__WEEX__) {
       ...
      } else {
      // createChildren会继续调用createElm函数创建
        createChildren(vnode, children, insertedVnodeQueue)
        if (isDef(data)) {
        // vnode里面的data存放该标签绑定的事件，此方法是给指定的标签绑定对应的方法
          invokeCreateHooks(vnode, insertedVnodeQueue)
        }
        // 当createChildren循环完成以后，会插入到执行的父级别标签
        // 当所有循环都走完insert会把创建好的dom结构插入到dom中
        insert(parentElm, vnode.elm, refElm)
      }

      if (process.env.NODE_ENV !== 'production' && data && data.pre) {
        inPre--
      }

    } else if (isTrue(vnode.isComment)) {
    // 创建注释
      vnode.elm = nodeOps.createComment(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    } else {
      vnode.elm = nodeOps.createTextNode(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    }
  }
```
  
总结：
1. 我们以2篇文章的篇幅把vue从初始化到最后生成页面，以及如何依赖收集简单阐述了一下
2. 其中需要简单说明一下的是，我们在方法里面

```
this.name = '4'
this.nameCopy = '3'
```
这么赋值的时候，是不是会重复连着两次更新页面ui呢？
其实是不会的，每一个watcher有一个唯一id，当我调用this.name的时候，会把watcher对应的cb函数放到callback的数组内，然后放到eventloop最后面执行（通过promise/setTimeout/MutationObserver实现的).接着执行this.nameCopy的时候，他会首先检查watcher id是否已经存在，如果存在则不会继续push到callback数组内。当两个赋值操作完成以后，才会循环执行callback里面的函数，以当前为例子，也就是调用vm.__update__(vm._render(), false)函数
3. 接着会继续补充一些重点api的文档，会以短文的形式发出，以上


