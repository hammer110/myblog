---
title: webpack import()动态引入
date: 2018-06-08 09:59:15
---

最近由于我司的架构迁移到spa，所以一些组件改为异步加载，但是这个时候build出来的js文件的名字都是以id命名的，以下为解决这个问题的思路：
 <!-- more -->
 
#### 1. 组件的引入改为异步

首先组件需要异步，并且在import方法内加上注释webpackChunkName：filename ***如果出现两个以上异步组件的webpackChunkName一样，那么这两个组件就打包到一个js文件内***

```
// 以vue-router为例

{
  path: '/home',
  name: 'Home',
  component: () => import(/* webpackChunkName: "Home" */ './Home')
}

```

#### 2. webpack打包配置

vue-cli默认chunkFilename取的是id，改为name

```
output: {
    path: config.build.assetsRoot,
    filename: utils.assetsPath('js/[name].[chunkhash].js'),
    chunkFilename: utils.assetsPath('js/[name].[chunkhash].js')
}

```

#### 3. 其他
一般情况1，2配置完成以后重新build文件名字就会变成以name命名的，但是我司用的vue-cli生成的模版，还需要修改.babelrc配置文件

```
{
....
 "comments": true // 默认为false，false会把所有的注释都编译掉
...
}
```


