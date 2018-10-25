---
title: vue-router源码分析
date: 2018-05-23 17:11:15
---

 现在网上vue-router源码分析的文章已经很多啦，为什么要写这一篇文章，主要是为了加深印象，毕竟别人的总归是别人的
 
 <!-- more -->
#### Vue.use(vueRouter)
这个阶段大致做了3件事
1. vue的原型链上挂载两个对象 `$router，$route`
2. 生成全局的mixin,挂载到每个组件中，这段代码很重要，赋值完成以后，能渲染到页面全都指望着这段代码啦
3. 注册全局组件 `router-view, router-link`

#### 初始化 `new VueRouter(routes)`
 
1. mode: vue-router的mode分为 `"hash" | "history" | "abstract"` 3种模式，我司目前的vue项目都启用的hash模式，他会根据不同的模式，new不同的示例

```
switch (mode) {
 case 'history':
   this.history = new HTML5History(this, options.base)
   break
 case 'hash':
   this.history = new HashHistory(this, options.base, this.fallback)
   break
 case 'abstract':
   this.history = new AbstractHistory(this, options.base)
   break
 default:
   if (process.env.NODE_ENV !== 'production') {
     assert(false, `invalid mode: ${mode}`)
   }
}
```
2.钩子函数：`beforeEach, afterEach`

  ```
      beforeEach (fn: Function) {
        this.beforeHooks.push(fn)
      }
    
      afterEach (fn: Function) {
        this.afterHooks.push(fn)
      }
  ```
  通过分析源代码可以看出，全局的钩子函数可以定义多个，他本质是存储到一个数组中
3. 在init函数中 **注册cb回调函数**，请注意cb函数很重要，新的路由信息，赋值都在cb函数内进行的

```
history.listen(route => {
 this.apps.forEach((app) => {
   app._route = route
 })
})
```
4. 监听hash值的变化

```
  if (history instanceof HTML5History) {
      history.transitionTo(history.getCurrentLocation())
    } else if (history instanceof HashHistory) {
      const setupHashListener = () => {
        history.setupListeners()
      }
      history.transitionTo(
        history.getCurrentLocation(),
        setupHashListener,
        setupHashListener
      )
}

// history.setupListeners
window.addEventListener('hashchange', () => {
     if (!ensureSlash()) {
       return
     }
     this.transitionTo(getHash(), route => {
       replaceHash(route.fullPath)
     })
})
```
上述代码很简单，就是在初始化的时候，在window对象上绑定 hashchange的事件，监听路由切换。不过上面代码还有一个重点就是在初始化的时候执行了 this.transitionTo方法


#### 跳转路由 transitionTo

```
transitionTo (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    const route = this.router.match(location, this.current)
    this.confirmTransition(route, () => {
      this.updateRoute(route)
      onComplete && onComplete(route)
      this.ensureURL()

      // fire ready cbs once
      if (!this.ready) {
        this.ready = true
        this.readyCbs.forEach(cb => {
          cb(route)
        })
      }
    }, onAbort)
  }
```
transitionTo的代码很简单，就是生成一个路由对象**route**, 这个route的数据结构，可以参考全局钩子函数内那个to的数据结构，和这个一样

#### 确定路由跳转 confirmTransition

confirmTransition 接受3个参数第一个是当前的路由对象，第二个是成功回调函数onComplete，第三个是失败回调函数onAbort

1. 生成一个事件队列

```
const queue: Array<?NavigationGuard> = [].concat(
 // in-component leave guards
 extractLeaveGuards(deactivated),
 // global before hooks
 this.router.beforeHooks,
 // in-component update hooks
 extractUpdateHooks(updated),
 // in-config enter guards
 activated.map(m => m.beforeEnter),
 // async components
 resolveAsyncComponents(activated)
)
```
2. 循环执行，当循环完成以后，执行onComplete

```
runQueue(queue, iterator, () => {
 const postEnterCbs = []
 const isValid = () => this.current === route
 const enterGuards = extractEnterGuards(activated, postEnterCbs, isValid)
 // wait until async components are resolved before
 // extracting in-component enter guards
 runQueue(enterGuards, iterator, () => {
   if (this.pending !== route) {
     return abort()
   }
   this.pending = null
   onComplete(route)
   if (this.router.app) {
     this.router.app.$nextTick(() => {
       postEnterCbs.forEach(cb => cb())
     })
   }
 })
})
```

3. confirmTransition onComplete回调函数核心代码

```
 // 渲染页面
 this.updateRoute(route)
 // replace路由name
 onComplete && onComplete(route)

```
4. updateRoute

```
updateRoute (route: Route) {
    const prev = this.current
    this.current = route
    this.cb && this.cb(route)
    this.router.afterHooks.forEach(hook => {
      hook && hook(route, prev)
    })
  }
```
updateRouter this.cb就是我们在 **初始化 第三3点的函数**
执行完this.cb以后就会执行，我们定义在afterEach里面函数
 
 5. this.cb函数,把当前组件更新到app._route上，
 
 ```
 this.apps.forEach((app) => {
    app._route = route
  })
 ```
当这里赋值的时候就会出发 router-view组件内的render函数，把最新的组件渲染进去

6. app._route数据更新以后是怎么通知router-view的，这就是涉及到了 `Vue.util.defineReactive(this, '_route', this._router.history.current)` 这个函数由于官网没有，经过查看vue源码，就是依赖于defineProperty的set 函数，当数据更新以后，就会出发底层的通知函数，然后触发router-view的render函数重新渲染


#### router-link
他本质上是vue一个全局的组件，然后通过你传给组件的值然后渲染出来不同的标签

```
if (this.tag === 'a') {
 data.on = on
 data.attrs = { href }
} else {
 // find the first <a> child and apply listener and href
 const a = findAnchor(this.$slots.default)
 if (a) {
   // in case the <a> is a static node
   a.isStatic = false
   const extend = _Vue.util.extend
   const aData = a.data = extend({}, a.data)
   aData.on = on
   const aAttrs = a.data.attrs = extend({}, a.data.attrs)
   aAttrs.href = href
 } else {
   // doesn't have <a> child, apply listener to self
   data.on = on
 }
}
```

这块代码是根据你传过来的tag值来区分是a标签还是其他标签，然后是给这个标签绑定点击事件还是直接添加href属性

### 2017-09-04更新
#### 路由改变的时候是怎么重新调用router-view组件的render方法的

上面已经大致梳理了vue路由系统的流程，这次我们简单介绍一下当app_route = route的时候，vue是怎么触发render函数的。
需要搞明白这个问题，我们需要研究一下vue的源码。在vue initData方法内，对组件data下的所有属性用defineReactive方法包装了一层

<pre>
/**
 * Define a reactive property on an Object.
 */
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: Function
) {
  const dep = new Dep()

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set

  let childOb = observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
        }
        if (Array.isArray(value)) {
          dependArray(value)
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
      childOb = observe(newVal)
      dep.notify()
    }
  })
}
</pre>

这个属性的所有watcher(watchers: 也就是当这个值改变的时候，会触发哪些cb函数)，都存到Dep这个函数的subs数组内。tips: 每个数组都有自己一个独有的dep实例
那么vue是怎么收集这个属性对应的所有watcher的呢？我们在router-view的render函数内获取了$route

<pre>
// src/components/view.js
render (h, { props, children, parent, data }) {
    data.routerView = true

    const name = props.name
    
    // 重点部分start
    const route = parent.$route
    // 重点部分end
    const cache = parent._routerViewCache || (parent._routerViewCache = {})

    // determine current view depth, also check to see if the tree
    // has been toggled inactive but kept-alive.
    let depth = 0
    let inactive = false
    while (parent) {
      if (parent.$vnode && parent.$vnode.data.routerView) {
        depth++
      }
      if (parent._inactive) {
        inactive = true
      }
      parent = parent.$parent
    }
    data.routerViewDepth = depth
    
    ......
})
</pre>

 const route = parent.$route 这个是重点，当你这么获取值的时候他会触发下面的get函数
 
<pre>
  // src/install.js
  Object.defineProperty(Vue.prototype, "$route", {
    get () { return this.$root._route }
  })
  Vue.mixin({
    beforeCreate () {
      if (this.$options.router) {
        this._router = this.$options.router
        this._router.init(this)
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      }
    }
  })
 </pre>
 
 别停，他接着会触发 ***Vue.util.defineReactive(this, '_route', this._router.history.current)*** 这个时候他又会触发这个属性对应的getter方法，这次触发getter方法，会把这个watcher存储到dep的subs内
 

回到开头的时候当我们切换路由app.route = route, 路由更新以后，就会触发_route对应的set函数，接着调用dep.notify()函数，循环执行subs数组内的所有cb函数。tips: 肯定会有同学问，这个情况下cb函数是什么？源码如下

<pre>
  vm._update(vm._render(), hydrating);
</pre>

调用_update方法，然后重新渲染router-view这个组件。以上，就是vue-router整体一个流程

#### 在组件内的钩子函数内，不手动调用next函数，为什么不会渲染页面／跳转下一个页面

这个问题，是在帮助别人解决相关问题的时候研究的，现总结如下，以beforeRouteLeave为例

<pre>
// base.js confirmTransition方法

const queue: Array<?NavigationGuard> = [].concat(
 // beforeRouteEnter存储到到这里
 extractLeaveGuards(deactivated),
 // global before hooks
 this.router.beforeHooks,
 // in-component update hooks
 extractUpdateHooks(updated),
 // in-config enter guards
 activated.map(m => m.beforeEnter),
 // async components
 resolveAsyncComponents(activated)
)

// 循环执行queue数组内的钩子函数

runQueue(queue, iterator, () => {
      const postEnterCbs = []
      const isValid = () => this.current === route
      const enterGuards = extractEnterGuards(activated, postEnterCbs, isValid)
      // wait until async components are resolved before
      // extracting in-component enter guards
      runQueue(enterGuards, iterator, () => {
        if (this.pending !== route) {
          return abort()
        }
        this.pending = null
        onComplete(route)
        if (this.router.app) {
          this.router.app.$nextTick(() => {
            postEnterCbs.forEach(cb => cb())
          })
        }
      })
    })
    
    this.pending = route
    const iterator = (hook: NavigationGuard, next) => {
      if (this.pending !== route) {
        return abort()
      }
      ／/ 
      hook(route, current, (to: any) => {
      
        if (to === false) {
          // next(false) -> abort navigation, ensure current URL
          this.ensureURL(true)
          abort()
        } else if (typeof to === 'string' || typeof to === 'object') {
          // next('/') or next({ path: '/' }) -> redirect
          (typeof to === 'object' && to.replace) ? this.replace(to) : this.push(to)
          abort()
        } else {
          // confirm transition and pass on the value
          next(to)
        }
      })
    }
    
</pre>

然后我们看一下 runQueue 方法是怎么实现的

<pre>
  export function runQueue (queue: Array<?NavigationGuard>, fn: Function, cb: Function) {
  const step = index => {
    if (index >= queue.length) {
      cb()
    } else {
      if (queue[index]) {
        fn(queue[index], () => {
          step(index + 1)
        })
      } else {
        step(index + 1)
      }
    }
  }
  step(0)
}
</pre>

现在开始梳理流程，如果我们在组件内的beforeRouteLeave内不手动调用next方法的话，不会触发hook的第三个函数，如果这个函数不会触发，也就不会调用iterator方法传过来的next方法。如果这个方法也不会触发的话，那么相当于runQueue方法内的 step(index + 1）方法不会触发，那么递归到此就停止啦。所以页面就卡死在这里。tips: hook, iterator, runQueue等方法请在上文的代码中找

#### 总结：
1. 通过学习vue-router源码，大致了解vue路由的大致运行原理
2. vue-router 还有自己的缓存系统，猜测是根据你是否配置keep-alive来进行缓存的
3. 由于经验有限，可能有的描述不太严谨，请多多指教


