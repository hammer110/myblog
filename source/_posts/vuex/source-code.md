---
title: vuex 源码解析
date: 2018-05-23 17:13:15
---

对vuex源码进行了一个简单的梳理
<!-- more -->
#### store初始化的过程
1. Vue.use(Vuex)
初始化的过程
调用install --> applyMixin --> Vue.mixin --> 往组件内this对象上挂载$store

<pre>
const options = this.$options
if (options.store) {
 // app.vue this上挂载$store对象
 this.$store = options.store
} else if (options.parent && options.parent.$store) {
 // 组件内的this对象上挂载$store对象 
 this.$store = options.parent.$store
}
</pre>

2.&nbsp;new 一个Vuex.store的实例对象, 传入配置参数

```
const debug = process.env.NODE_ENV !== 'production'
const store = new Vuex.Store({
  // 所有action的集合
  actions,
  // 所有gettrs的集合
  getters,
  // 对应的模块
  modules: {
    api
  },
  // 是否开启严格模式，线上会关掉，其余环境都会开启
  strict: debug
})
```

3. Store的constructor进行初始化，注册action, mutation

<pre>
function installModule (store, rootState, path, module, hot) {
  const isRoot = !path.length
  // 当namespaced=true 的时候，其便成为命名空间模块。当模块被注册后，
  // 它的所有 getter、action 及 mutation 都会自动根据模块注册的路径调整命名
  const namespace = store._modules.getNamespace(path)

  // register in namespace map
  if (namespace) {
    store._modulesNamespaceMap[namespace] = module
  }

  // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      Vue.set(parentState, moduleName, module.state)
    })
  }
// 如果namespace是‘’，那么makeLocalContext返回store原型上挂载的对应的dispatch, commit，
// 然后利用defineProperties，监听getter和state的改变
  const local = module.context = makeLocalContext(store, namespace, path)
// 循环注册mutations
  module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key
    registerMutation(store, namespacedType, mutation, local)
  })
// 循环注册actions
  module.forEachAction((action, key) => {
    const namespacedType = namespace + key
    registerAction(store, namespacedType, action, local)
  })
// 循环注册getters
  module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key
    registerGetter(store, namespacedType, getter, local)
  })
// 循环注册module
  module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot)
  })
}

</pre>

4.循环注册mutation

<pre>
function registerMutation (store, type, handler, local) {
// 第一次进来的时候state还是undefined，等执行完 resetStoreVM这个方法以后才能够初始化，
// 其实state就是获取的this._vm._data.$$state
  var entry = store._mutations[type] || (store._mutations[type] = []);
  entry.push(function wrappedMutationHandler (payload) {
  // 调用mutation对应的函数，第一个参数默认传入local.state,
  // 如果只是单纯的从上下文来看，这个时候的local.state是指的全局的state，
  // 但是使用的时候，state就是指的当前这个module下的state，是因为在installModule方法
  // 最后一行重新调用installModule, 这个时候传入的path有值，
  // 相当于每一个module都在本地维护一个自己的local.state,
  //这个local.state只是当前模块的state
    handler(local.state, payload);
  });
}
</pre>

5.action为什么第一个参数是store对象

<pre>
  function registerAction (store, type, handler, local) {
  const entry = store._actions[type] || (store._actions[type] = [])
  entry.push(function wrappedActionHandler (payload, cb) {
    let res = handler({
      dispatch: local.dispatch,
      commit: local.commit,
      getters: local.getters,
      state: local.state,
      rootGetters: store.getters,
      rootState: store.state
    }, payload, cb)
    if (!isPromise(res)) {
      res = Promise.resolve(res)
    }
    if (store._devtoolHook) {
      return res.catch(err => {
        store._devtoolHook.emit('vuex:error', err)
        throw err
      })
    } else {
      return res
    }
  })
}
</pre>

handler代表的是你定义的action函数，第一个参数是在注册的时候自动创建一个对象，里面包含上述参数， vuex的action肯定返回一个promise对象回去

6.getters是怎么注册的

调用getters的方法目前有两种方式：
1).直接通过store.getters

<pre>
function resetStoreVM (store, state, hot) {
  const oldVm = store._vm
  // bind store public getters
  store.getters = {}
  // _wrappedGetters是配置的所有getters
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  forEachValue(wrappedGetters, (fn, key) => {
    // fn代表的是你定义的getters函数
    computed[key] = () => fn(store)
    Object.defineProperty(store.getters, key, {
     // 获取对应的getters的时候直接把挂载到vue实例对象上的方法返回
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
  })
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
  // 开启严格模式，禁止用户直接修改vuex的数据
  // 开启严格模式以后，每次修改state的时候，发送commit的时
  // 会把store._commiting改为true, 调用完成mutation以后再改为false
  // 当state的数据改变的时候，会先判断commiting的状态
  // 如果为false的话，直接throw new Error出来
  // 如果开启严格模式，那么就不能在mutation里面封装异步函数
  // 这种情况会判定为手动修改state的值
  if (store.strict) {
    enableStrictMode(store)
  }
  // 如果内存中已经存在oldVm删掉->避免频繁更新或者注册，导致内存中出现多个oldVm
  if (oldVm) {
    if (hot) {
      // dispatch changes in all subscribed watchers
      // to force getter re-evaluation for hot reloading.
      store._withCommit(() => {
        oldVm._data.$$state = null
      })
    }
    Vue.nextTick(() => oldVm.$destroy())
  }
}
</pre>

**2017-8-28 update**
关于为什么要把state传到vue的实例里面，我的理解是利用vue封装好的defineProperty+pub/sub的类，监听数据的改变，触发对应的watchers。并且页面中获取state值的方法都是直接或者间接从store._vm._data.$$state.state 这种方式获取的, 然后当数据更改的时候就能触发对应字段的set方法，然后通知各个watcher

2).在动态计算属性里面配置

这里借助的是mapGetters方法

<pre>
export const mapGetters = normalizeNamespace((namespace, getters) => {
  const res = {}
  normalizeMap(getters).forEach(({ key, val }) => {
    val = namespace + val
    res[key] = function mappedGetter () {
      if (namespace && !getModuleByNamespace(this.$store, 'mapGetters', namespace)) {
        return
      }
      if (!(val in this.$store.getters)) {
        console.error(`[vuex] unknown getter: ${val}`)
        return
      }
      return this.$store.getters[val]
    }
    // mark vuex getter for devtools
    res[key].vuex = true
  })
  return res
})
</pre>

其实本质上也是借助于this.$store.getters方法，返回对应的函数回去，动态计算属性会执行这个函数，然后return回去对应的state


7.dispatch

```
dispatch (_type, _payload) {
    // check object-style dispatch
    const {
      type,
      payload
    } = unifyObjectStyle(_type, _payload)

    const entry = this._actions[type]
    if (!entry) {
      console.error(`[vuex] unknown action type: ${type}`)
      return
    }
    return entry.length > 1
      ? Promise.all(entry.map(handler => handler(payload)))
      : entry[0](payload)
  }
```
同一个type对应1个函数的时候直接执行，把payload参数传进去。当对应两个或者以上的函数时，是用的promise.all方法进行调用的


8.vuex插件

合并devtool和自定义插件，然后循环调用，把this(store)传入插件内

```
plugins.concat(devtoolPlugin).forEach(plugin => plugin(this))
```

一般插件都是这么定义的

<pre>
const myPlugin = store => {
  // 当 store 初始化后调用
  store.subscribe((mutation, state) => {
    // 每次 mutation 之后调用
    // mutation 的格式为 { type, payload }
  })
}
</pre>

初始化调用就不多说啦，那么store.subscribe怎么实现，每次commit的时候，就触发该方法

<pre>
subscribe (fn) {
    const subs = this._subscribers
    if (subs.indexOf(fn) < 0) {
      subs.push(fn)
    }
    return () => {
      const i = subs.indexOf(fn)
      if (i > -1) {
        subs.splice(i, 1)
      }
    }
}
</pre>

首先加载插件的时候就会执行subscribe方法，然后把回调函数存储到_subscribers数组内，然后在commit的方法内，当mutation回调函数执行完成以后，循环执行存储到_subscribers数组内的方法

```
store.js
method: commit

this._subscribers.forEach(sub => sub(mutation, this.state))
```


#### 总结

1. 经过对vuex源码的分析，更加深入的了解了vuex执行原理
2. 经过查看vuex.esm.js对对象继承，this执行等js基础执行，有了更加深入的了解
3. 文章中如果有所纰漏，敬请指出 
4. 下一步是看vue的源码，不过vue相关文章应该会分章节，分模块的写，不会一次写完，敬请期待

