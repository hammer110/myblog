---
title: 关于async和await一个面试题
date: 2018-09-30 12:04:00
---

昨天在一个技术群划水，看到一个哥们问一个技术题，感觉特别有意思，mark一下

<!-- more -->

### 面试题
main的打印顺序

```
let docs = [{}, {}, {}]
docs.forEach(async function (doc) {
	await post(doc)
	console.log("main")
})
function post () {
	return new Promise((resolve) => {
		setTimeout(resolve, 1000)
	})
}
```

结果是一秒以后并行打印3次main，但是他想要实现的效果是每隔一秒打印一次main

第一次看到这个问题我也是一脸懵，后来经过大神讲解，简单了解了一下前因后果，mark一下

### 为什么会并行

按照上述写法，其实是本质形成了3个async作用域，3个await是在3个不同作用域内，所以会并发执行，上述代码的效果类似于下图

```
// 1
async function (doc) {
    await post(doc)
    console.log("main")
}
// 2
async function (doc) {
    await post(doc)
    console.log("main")
}
// 3
async function (doc) {
    await post(doc)
    console.log("main")
}
```

### 串行实现方法

如果想要实现串行，需要实现类似于如下效果

```
async function (doc) {
    // 1
    await post(doc)
    console.log("main")
    // 2
    await post(doc)
    console.log("main")
    // 3
    await post(doc)
    console.log("main")
}
```

通过这个思路可以得到代码实现如下(某大神写的)

```
async function run () {
  for (let i = 0; i < docs.length; i++) {
    await post(docs[i])
    console.log("main")
  }
}
```
 
第一次看到这个实现方式的时候比较疑惑，在for循环的时候用let定义i，他会形成独立作用域，一般形成独立作用域都是把for循环内的内容用IIFE包裹起来，然后i传入到IIFE内(本质上就是利用闭包的概念)

```
async function run () {
  for (let i = 0; i < docs.length; i++) {
    (function (j) {
       await post(docs[j])
       console.log("main")
    })(i)
  }
}
```
我以为babel会编译成这样，按道理就会***报await必须定义在async函数内的error***，因为async和await中间还隔着一个函数

经过去[babel](https://babeljs.io/repl)官网验证，发现自己想错了，最后编译出来的代码如下

```
for (var i = 0; i < docs.length; i++) {
    post(docs[i]);
    console.log("main");
}
```

经过上述论证，得出一个不严谨的结论，如果for循环内是一个***异步函数***，***并且***这个函数需要引用到for循环内定义的索引，那么babel就会把它用IIFE包裹起来

### 总结

技术之路何其漫长


