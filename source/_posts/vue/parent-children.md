---
title: vue源码系列-番外篇(子父组件通信机制) 
date: 2018-05-23 17:16:15
---

众所周知vue子父组件数据传输是通过prop或者vuex，但是有一些场景，需要子组件调用父组件内定义的方法，这个时候我们今天的主角“$on, $emit”，就该登场了

<!-- more -->
#### 1. 一个简单的demo
    
```
<div id="app">
 <div>{{name}}</div>
 <my-component v-on:update="update"></my-component>
</div>

```

```
var Child = {
 template: '<div>A custom component! <div @click="triggerInfo">send btn</div></div>',
 methods: {
   triggerInfo () {
     this.$emit('update')
   }
 },
}
  new Vue({
    el: '#app',
    data: {
      name: 'hammer'
    },
    components: {
     'my-component': Child
   },
   methods: {
     update () {
       console.log('render')
     }
   },
    created () {
      setTimeout(() => {
        this.name = 'liujinjian'
      }, 3000)
    }
  })
```

上述代码是一个很简单的demo，就是点击子组件的‘send btn’ 按钮，调用 ‘this.$emit('update')’，触发父组件的 methods里面定义的update方法

#### 2. vue组件内的methods是怎么初始化的

上一篇vue源码分析01的文章中，我们已经简单介绍了在初始化组件的时候会调用initMethods方法, 把methods里面定义的方法用bind方法包裹一层，然后挂载到vm实例上

```
export function bind (fn: Function, ctx: Object): Function {
// ctx为this对象
  function boundFn (a) {
    const l: number = arguments.length
    return l
      ? l > 1
        ? fn.apply(ctx, arguments)
        : fn.call(ctx, a)
      : fn.call(ctx)
  }
  // record original fn length
  boundFn._length = fn.length
  return boundFn
}
```

#### 3. 监听函数，父组件是怎么传入的

需要搞明白这个，我们就需要搞明白我们定义在html里面的模版最后会编译成什么样子，以上文demo为例，我们在初始化以后会调用$mount方法，在这个方法内会调用baseCompile方法编译html，最后返回一个ast抽象语法树，并且还有一个render方法

```
// 执行 render.toString() 打印结果如下
"function anonymous() {
with(this){return _c('div',{attrs:{"id":"app"}},[_c('div',[_v(_s(name))]),_v(" "),_c('my-component',{on:{"update":update}})],1)}
}"
```

这段代码利用with特性. ***tips:我们初始化的时候，把data和methods等属性和方法都直接挂载this上，在这个方法anonymous内的变量，会从this上直接取到赋值，或者传给子组件***


#### 4. 生成vnode

调用_render方法
vnode = render.call(vm._renderProxy, vm.$createElement);

vnode的children属性下面找到最后一个对象下面componentOptions下面的listeners就是你从父组件传给子组件的监听方法

#### 5. 渲染子组件

当我们渲染子组件的时候，我们会重新调用this._init方法进行初始化，由于是子组件，会调用createComponentInstanceForVnode会把一些参数赋值给option

```
var options = {
    _isComponent: true,
    parent: parent,
    propsData: vnodeComponentOptions.propsData,
    _componentTag: vnodeComponentOptions.tag,
    _parentVnode: vnode,
    // listeners赋值给了_parentListeners
    _parentListeners: vnodeComponentOptions.listeners,
    _renderChildren: vnodeComponentOptions.children,
    _parentElm: parentElm || null,
    _refElm: refElm || null
  };
```
接着调用_init方法，把该option传入

#### 6. 子组件是怎么存储该函数的

在initEvents方法内

```
function initEvents (vm) {
  vm._events = Object.create(null);
  vm._hasHookEvent = false;
  // init parent attached events
  var listeners = vm.$options._parentListeners;
  if (listeners) {
    updateComponentListeners(vm, listeners);
  }
}
```

如果_parentListeners存在会依次调用updateComponentListeners --> updateListeners --> add方法

```
／/ add function
function add (event, fn, once$$1) {
  if (once$$1) {
    target.$once(event, fn);
  } else {
    target.$on(event, fn);
  }
}
```

```
接着触发“target.$on”方法，把父组件传过来的经过bind包装的update方法存储到this._events对象内，当我们调用 “$emit” 方法的时候，他会去this._events里面找到对应type的方法，然后调用
```

以上，就是vue子父组件通信的一个大致流程


#### 总结

1. vue的通讯就是基于一个挂载到vm实例上的属性来做的，监听方法会往events里面存储，emit的时候会找到对应方法然后调用
2. 刚开始被一个点给迷惑了，就是子父组件都有自己的独立作用域，那么我在子组件调用对应的方法的时候，this应该指向子组件，那么在这个监听函数内，通过this.methodName是取不到对应的方法或者data里面的属性的。那么他是怎么做到的呢？其实是我一直在强调的，监听函数在父组件让bind包裹了一层，并且他还是一个闭包函数，所以在子组件调用该方法的时候，该方法取到的ctx还是初始化的时候传进来的ctx也就是父组件的this.


