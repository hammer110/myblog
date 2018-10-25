---
title: 关于action的一些想法
date: 2018-05-23 17:08:15
---

在使用vuex的场景中，我们会经常用到发送action改变store内state值的场景，如下例子
<!-- more -->
```
  import { mapActions } from 'vuex'
  
  methods: {
    ...mapAction(['updateUserInfo'])
  }
  created () {
    this.updateUserInfo({'token': 'abcedfeddtss'})
    .then(() => {
       console.log(this.userInfo.token)
    })
  }
  
  // action:updateUserInfo的实现
  export const updateUserInfo = ({ commit }, ...args) => {
    commit('UPDATEUSERINFO', ...args)
  }
```
这是一个简单的例子，但是注意的是

1. 发送action为什么可以在后面用.then() -> 也就是为什么action会返回一个promise对象
2. action里面明明是同步的为什么还要用.then


### vuex的初始化过程

当我们初始化vuex实例的时候，会把action,getters,state都传入到构造函数中

```
 const store = new Vuex.Store({
  actions,
  getters,
  modules: {
    userInfo
  }
})
```

在这一步的时候vuex会对所有的action进行一步包装的过程

```
function registerAction (store, type, handler, local) {
  const entry = store._actions[type] || (store._actions[type] = [])
  entry.push(function wrappedActionHandler (payload, cb) {
    let res = handler.call(store, {
      dispatch: local.dispatch,
      commit: local.commit,
      getters: local.getters,
      state: local.state,
      rootGetters: store.getters,
      rootState: store.state
    }, payload, cb)
    // 这里是重点
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
```
如果res返回的不是一个promise对象，会手动调用promise对他进行包装。这就是为什么你可以发送完action以后可以用.then()。否则没有这一步，你会直接报错'then is not a function'

***同时从这里也能明白，我们在使用action方法的时候明明第一个参数是我们想要处理的数据，但是action方法内接收到的第一个参数是store对象，归根到底还是因为vuex对action进行了包装***


### 关于是否需要用.then的考虑

在上面那个修改token值的例子中，其实用不用.then都是一样的

```
  // demo1
  this.updateUserInfo({'token': 'abcedfeddtss'})
    .then(() => {
       console.log(this.userInfo.token)
    })
 // demo2
  this.updateUserInfo({'token': 'abcedfeddtss'})
   console.log(this.userInfo.token)
```

在上述例子中demo1 === demo2

但是上述例子内有一个坑，那就是promise.then()内的回调函数是异步，举例说明

```
  
  this.updateUserInfo({'step': 'step one'})
    .then(() => {
       console.log(this.userInfo.step)
    })
  console.log('step', 'two')
  
  // step two
  // step one
```

***重点promise方法内不是异步的，他的.then回调函数内才是异步的***

```
console.log(1)
new promise((r, j) => {
  console.log(2)
  r()
}).then(() => {
console.log(3)
})
console.log(4)
setTimeout(() => {
 console.log(5)
}, 0)

// 1
// 2
// 4
// 3
// 5
```
从这个例子内可以看出.then是异步的，他会放到事件队列后面执行。

关于setTimeou和promise.then谁先执行的问题，不在这里展开，可以自行百度一下

#### 最后总结
在我司大部分场景中demo1和demo2的用法，是可以混着用的，不过还是应该明白他的区别。关于当时为什么会直接选用demo1的写法，我感觉这样会在直观上给人一种，后面的逻辑需要等到action执行完以后才会执行。***（虽然用不用效果都一样）***



