---
title: 记一次开发webpack loader的过程
date: 2018-08-09 15:30:15
---

最近同事提出了一个需求，想要执行build的时候直接过滤到某部分js代码，因为这部分js代码线上用不到，下面是实现这个需求的一个大致过程
 <!-- more -->
### 选取方案

#### tree-shaking
第一次想到的就是基于webpack的tree-shaking来实现这个需求，但是实现过程中发现，tree-shaking有一些场景满足不了，先让我们看一下它满足的场景

***场景1***

```
// util.js
export const a = function () {}
export const b = function () {}

// a.js
import { a } from './util'
a()
```
这种情况方法b编译阶段会删掉

***场景2***
```
// util.js
export const a = function () {}
function test () {}
// a.js
import { a } from './util'
a()
```

这种情况方法test编译阶段会删掉

***不适用的场景***

```
// util.js
export default class Test {
  check () {}
}
// a.js
import Test from './util.js'
new Test()
```
这种情况Test类内的check方法就不会被过滤掉，通过上述几个demo，我们可以得到一个不严谨的结论

***当我对外抛出的function、class内即便有部分方法没有引用到，tree-shaking也不会过滤掉***

#### plugin
tree-shaking失败以后，另外想到的就是写一个webpack plugin，但是通过查看文档没有发现操作js源数据的api（很有可能我没发现）

#### loader
loader本质上就是一个函数，把匹配到的文件内容，比如js，转成字符串，在loader内对这个字符串进行压缩替换等操作后再返回，让后续的loader继续操作

定义一个loader函数

```
// loaders/ignore-content-loader.js
const reg = /\/\* ignore\ start \*\/[\s\S]*?\/\* ignore\ end \*\//g
module.exports =  function(source) {
// 根据正则匹配到/* ingore start */ /* ingore end */之间的内容，然后通过replace置为空
  return source.replace(reg, '');
};
```
webpack.base.conf.js

```
// 在babel-loader后面新增我们自定义的loader ignore-content-loader
module.exports = {
  module: {
    rules: [
      {
        test: /\.js$/,
        loader: ['babel-loader', 'ignore-content-loader'],
        include: [resolve('fe'), resolve('test'), resolve('node_modules/webpack-dev-server/client')]
      },
    ]
  },
  // 这个代码很重要，否则我们自定义的loader他会找不到
    resolveLoader: {
        // 告诉 webpack 该去那个目录下找 loader 模块
        modules: ['node_modules', path.resolve(__dirname, 'loaders')]
     }
}
```

### 总结
我们这个需求比较简单，loader提供的方法好多都没用到，不过通过这一次也算对loader做了一些梳理


### 参考文档
https://juejin.im/post/5a698a316fb9a01c9f5b9ca0
https://www.webpackjs.com/contribute/writing-a-loader/


