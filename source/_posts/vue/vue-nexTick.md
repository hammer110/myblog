---
title: vue $nextTick
date: 2018-07-07 16:37:15
---

在vue组件内，当改变data内的某个值时，这个值会造成dom结构的改变，这个时候我们想要获取改变后的dom结构，一般是使用$nextTick，那么它是怎么实现的呢？我们接下来分析

<!-- more -->

### 一个栗子
```
<template>
  <div class="flag" v-if="flag"></div>
</template>
<script>
  export default {
    data () {
      return {
        flag: false
      }
    },
    mounted () {
      this.flag = true
      console.log(document.querySelector('.flag')) // null
      // 方式1
      this.$nextTick(() => {
        console.log(document.querySelector('.flag')) // dom
      })
      // 方式2
      this.$nextTick().then(() => {
        console.log(document.querySelector('.flag')) // dom
      })
    }
  }
</script>
```
### $nextTick实现

通过上面的demo我们可以看到$nextTick内可以获取到dom元素，而它外面获取不到，并且$nextTick定义还有两种方式，那么我们看一下this.$nextTick怎么实现的

```
// this.$nextTick
Vue.prototype.$nextTick = function (fn) {
   return nextTick(fn, this)
};
```
$nextTick底层直接调用了nextTick方法

```
// nextTick
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  // 缓存回调函数到callBacks内，等后期直接循环callbacks，执行对应的函数
  callbacks.push(() => {
    if (cb) {
      try {
      // 直接执行cb函数
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
      // 当$nextTick没有传入回调函数，回直接返回一个promise对象
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
       if (useMacroTask) {
    // 目前没有分析出来那个流程会走这个分支
      macroTimerFunc()
    } else {
      microTimerFunc()
    }
  }
  // 如果没有传入回调函数 _resolve等于promise的resolve
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

当前情况会直接执行microTimerFunc函数

```
// microTimerFunc
var p = Promise.resolve();
microTimerFunc = function () {
// flushCallbacks事件队列最后面
    p.then(flushCallbacks);
    // 在有问题的UIWebViews中，Promise.then并没有完全破坏，
    // 但它可能会陷入一种奇怪的状态，即回调被推入任务队列，
    // 但队列没有被刷新，直到浏览器需要做其他工作，
    // 例如 处理一个计时器。 因此，我们可以通过添加空计时器来“强制”刷新微任务队列。
    if (isIOS) { setTimeout(noop); }
  };
// 循环执行callbacks数组内的函数
function flushCallbacks () {
  pending = false;
  var copies = callbacks.slice(0);
  callbacks.length = 0;
  for (var i = 0; i < copies.length; i++) {
    copies[i]();
  }
}
```
上述是$nextTick一个完整的执行过程，本质上就是nextTick内的回调函数放到事件队列最后面。通过Promise.resolve().then()

### microTimerFunc小插曲

microTimerFunc其实在vue初始化的过程中会根据当前运行环境给microTimerFunc赋值不同的调用方式

```
var microTimerFunc;
var macroTimerFunc;
var useMacroTask = false;

// Determine (macro) task defer implementation.
// Technically setImmediate should be the ideal choice, but it's only available
// in IE. The only polyfill that consistently queues the callback after all DOM
// events triggered in the same loop is by using MessageChannel.
/* istanbul ignore if */
// node、ie11、edge运行环境setImmediate才会存在
if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  macroTimerFunc = function () {
    setImmediate(flushCallbacks);
  };
  // 是否支持posemeeage
} else if (typeof MessageChannel !== 'undefined' && (
  isNative(MessageChannel) ||
  // PhantomJS
  MessageChannel.toString() === '[object MessageChannelConstructor]'
)) {
  var channel = new MessageChannel();
  var port = channel.port2;
  channel.port1.onmessage = flushCallbacks;
  macroTimerFunc = function () {
    port.postMessage(1);
  };
} else {
// 当上面都不支持的时候会利用settimeout放到时间队列最后面
  /* istanbul ignore next */
  macroTimerFunc = function () {
    setTimeout(flushCallbacks, 0);
  };
}
// 如果当前环境支持promise会走单独的定义函数，否则会吧上面定义的macroTimerFunc赋值给microTimerFunc
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  var p = Promise.resolve();
  microTimerFunc = function () {
    p.then(flushCallbacks);
    // 在有问题的UIWebViews中，Promise.then并没有完全破坏，
    // 但它可能会陷入一种奇怪的状态，即回调被推入任务队列，
    // 但队列没有被刷新，直到浏览器需要做其他工作，
    // 例如 处理一个计时器。 因此，我们可以通过添加空计时器来“强制”刷新微任务队列。
    if (isIOS) { setTimeout(noop); }
  };
} else {
  // fallback to macro
  microTimerFunc = macroTimerFunc;
}
```

### 为什么需要调用nextTick

在上文的demo中我们解释了nextTick实现的原理，本质上就是放到事件队列最后面，那么为什么上文中获取dom元素的代码非得放到队列最后面才可以获取成功呢？我们直接看核心代码

```
// this.flag = true的时候就会直接到该流程
function queueWatcher (watcher) {
// watcher对应的是this.flag改变以后需要调用的回调函数
// 当前是更新dom结构

// 每一个watcher有一个唯一ID
  var id = watcher.id;
  // 判断当前对象内是否已经包含了该watcher
  if (has[id] == null) {
    has[id] = true;
    // flushing代表当前是否在执行queue队列的过程中
    if (!flushing) {
    // 把当前的watcher放到队列内没有直接执行
      queue.push(watcher);
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      var i = queue.length - 1;
      while (i > index && queue[i].id > watcher.id) {
        i--;
      }
      queue.splice(i + 1, 0, watcher);
    }
    // queue the flush
    if (!waiting) {
      waiting = true;
      // nextTick上文已经说了，本质上就是把flushSchedulerQueue放到当前的事件队列最后面
      nextTick(flushSchedulerQueue);
    }
  }
}
// flushSchedulerQueue
function flushSchedulerQueue () {
  flushing = true;
  var watcher, id;

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  
  // queue内的watcher按照id的大小进行排序
  queue.sort(function (a, b) { return a.id - b.id; });

  // do not cache length because more watchers might be pushed
  // 循环执行watcher函数
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index];
    id = watcher.id;
    has[id] = null;
    // 执行回调函数
    watcher.run();
    // in dev build, check and stop circular updates.
    if ("development" !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1;
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? ("in watcher with expression \"" + (watcher.expression) + "\"")
              : "in a component render function."
          ),
          watcher.vm
        );
        break
      }
    }
  }

  // keep copies of post queues before resetting state
  var activatedQueue = activatedChildren.slice();
  var updatedQueue = queue.slice();
  // 重置queue、has等字段
  resetSchedulerState();

  // 执行active、update等回调函数
  callActivatedHooks(activatedQueue);
  callUpdatedHooks(updatedQueue);

  // 通知vue的Google插件
  if (devtools && config.devtools) {
    devtools.emit('flush');
  }
}

```

现在我们来梳理一下，当我们调用this.flag = true的时候，他不是直接执行，而是通过nextTick放到事件队列最后面，这也就是为什么我们必须通过$nextTick才能成功回去dom元素的原因。那么为什么vue要这么设计呢？让我们来看一下下面的场景

```
<template>
  <div class="flag" v-show="flag"></div>
  <div class="flag" v-show="flag2"></div>
</template>

export default {
  data () {
    return {
      flag: false,
      flag2: false
    }
  },
  mounted () {
    this.flag = true
    this.flag2 = true
  }
}
```
上述代码如果不按照现有的架构设计就会出现flag、flag2两次改值会造成同一个回调函数update执行两次，造成性能的浪费

### 总结

这一次总结还是挺有收获的
1. $nextTick有两种实现方式
2. nextTick实现方式改变


