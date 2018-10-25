---
title: 老项目构建架构迁移
date: 2018-08-07 17:30:15
---

目前把abc classroom构建架构整体迁移到了vue-cli

 <!-- more -->

### 目标:
1. 构建速度提升
2. 图片压缩的集成
3. 拥抱新技术
4. js，css包的瘦身 （以teacher.js部分为例，js包由原来的500kb，缩小到现在的310kb）

### 目录结构
原来的设想是重新创建一个新项目做迁移，但是考虑到代码迁移难度（主要在迁移阶段还有开发的需求）所以在原有的基础上checkout 一个新的分支来做这个事情，所以大部分目录结构都没动

### 基础配置

1. 新的构建流程引入了eslint，基于standard规范（本来想用airbnb，但是他识别import语法总是有问题）
2. babel配置文件，新增了element-ui和lodash按需加载的配置
3. jade模版都是通过jade命令自动编译成html的

### 公共库
公共库主要涉及到common.css和base.min.js，刚开始设想错了，以为公共库都是不会改变的，但是和别的同学沟通发现，这些库改动还是比较频繁的，所以就不能提前预编译出来，需要在每次build的时候动态提取。

js库有模块化(umd)和非模块化之分(jquery库之类的)，区别是模块化需要通过require, import等引入（当然最新的浏览器也支持直接通过script引入es6的模块），非模块一般通过script标签引入，对象直接挂载到window对象上，比如window.vue、window.jQuery等。abc项目原来的一些基础库(vue、vuex、farbic等都是通过第二种方式引入)，考虑到改动的代码尽量少，所以需要使用第二种方式，兼容现有业务逻辑

在base.min.js 把基础库引入进来，手动挂载到全局对象上

```
// fe/js/lib/base.min.js
import Vue from './vue.esm'
import Vuex from './vuex.esm'
import $ from 'jquery'
import './velocity'
import './velocity.ui.min.js'
import 'jquery-confirm'
import fabric from 'fabric'
window.$ = jQuery
window.Vue = Vue
window.Vuex = Vuex
window.fabric = fabric.fabric
```

配置文件做对应修改

```
// webpack.base.conf.js
module.exports = {
  plugins: [
    new webpack.ProvidePlugin({
      $: 'jquery',
      jQuery: 'jquery'
    })
  ],
  externals: {
    '$': 'window.$',
    'jQuery': 'window.$',
    'Snap': 'window.Snap',
    'Velocity': 'window.Velocity',
    'vue': 'window.Vue',
    'vuex': 'window.Vuex'
  }
}

```

上面webpack.base.conf.js里面externals配置至关重要，如果我不这么配置

```
// vue => node_modules/vue
import Vue from 'vue'
```

按照上述配置完成以后

```
// vue => fe/js/lib/base.min.js 里面的window.vue
import Vue from 'vue'
```

### css样式
由于该项目css部分是stylus和less混用的，less部分比较简单，但是stylus配置起来比较坑

##### 1. 引入nib这个stylus库的时候，由于他底层用到了display: box，但是编译的时候他会直接报错，让用display:flex，为了避免这种错误，可以在引入nib库之前先初始化一些配置

```
flex-version = flex
support-for-ie = false
vendor-prefixes = official
@import "nib"
```
##### 2. 改变原来nib引入方式

原来的引入方式webpack编译的时候识别不了

```
// old
@import "nib"
// new
@import "../../../../node_modules/nib/index";
```

##### 3. stylus内的css提到入口处引入

_classroom.styl文件内引入了jquery-confirm.min.css文件，webpack编译的时候识别不了，所以把该css提取出来放到teacher.js头部引入

```
// teacher.js
import "@/assets/css/common/lib/jquery-confirm.min.css"
```

##### 4. stylus引入方式改变
原来的架构是先在jade模版内引入一个teacher/index.css，等gulp把对应模块，比如老师对应的stylus编译成css以后再替换jade模版对应css的路径和名称。但是webpack打包css、img、js的时候必须在入口或者入口引入的文件内有依赖关系，所以需要把teacher/index.stylus放到入口文件内引入

```
// teacher.js
import "@/assets/css/classroom/teacher/index.styl"
```
build的时候他会把打出来的css注入到对应的html模版内

### 图片部分
图片部分修改的点都是在引入方式上
原来图片都是通过下面方式引入的，然后在gulp编译的时候会替换为绝对路径，由于构建架构调整，修改如下

```
background: url({{{img/cousr/tap.png}}});
```

##### css内图片引入

```
// old
background: url({{{'classroom/img/cursor/tap.png'}})
// new 
background: url('~img/cursor/tap.png')
```

***tips: 如果图片路径如果用到webpack的alias，那么img前面必须有～，否则打包出来的时候路径不对***

##### js内图片引入

```
// old
 this._canvas.freeDrawingCursor = 'url({{{classroom/img/cursor/pencial.png}}}), crosshair'
 
 // new 
  this._canvas.freeDrawingCursor = `url(${require('img/cursor/pencial.png')}), crosshair`
```

***tips: 引入方式改为require***

## topcenter和staticHost

topcenter和staticHost原来是通过express渲染模版的时候往模版里面注入的，由于现在架构已经抛弃了服务端渲染，所以所有从dom模版获取这两个字段的地方都改为从配置文件读取

```
// old
var topcenter = $('#topcenter').val()
var static_host = $('#staticHost').val()
// new 
import { topcenter, static_host } from '../../vf2e.config'
export const NET_DETECT = topcenter
export const HOSTS = window.location.pathname.indexOf('teacher') > -1 ? static_host.teacher.sayabc : static_host.student.sayabc

```

### 捕捉script标签错误

老的build架构 js引入和css引入是一个套路，原架构可以在script上手动加上onerrer捕捉该脚本的错误，但是现在script标签都是webpack自动注入进去的，那么就需要引入第三方webpack plugin来处理，webpack.prod.conf.js新增如下代码

```
const ScriptExtHtmlWebpackPlugin = require('script-ext-html-webpack-plugin')
new ScriptExtHtmlWebpackPlugin({
  custom: [
  {
    test: /\.js$/,
    attribute: 'onerror',
    value: 'window.logResErr(this)'
  }]
})

// build成功以后
<script type=text/javascript src=/static/js/teacher.a7fd26b5b19a67ec6ded.js onerror=window.logResErr(this)></script>
```

### webpack

webpack新增了test、pre命令，图片压缩、以及公共包抽取方式的改变

图片压缩
```
// webpack.base.conf.js
{
test: /\.(png|jpe?g|gif|svg)(\?.*)?$/,
use: [
  `url-loader?name=${utils.assetsPath("img/[name].[hash:7].[ext]")}&limit=10`,
  {
    loader: "image-webpack-loader",
    options: {
      mozjpeg: {
        progressive: false,
        quality: 80
      },
      // optipng.enabled: false will disable optipng
      optipng: {
        enabled: false
      },
      pngquant: {
        quality: "65-90",
        speed: 4
      },
      gifsicle: {
        interlaced: false
      }
    }
  }
]

}
```

publicPath修改

```
// webpack.base.conf.js
let assetsPublicPath = '/'
if (process.env.NODE_ENV === 'production') {
  assetsPublicPath = config['build'].assetsPublicPath
} else {
  assetsPublicPath = config[process.env.NODE_ENV].assetsPublicPath
}

module.exports = {
 output: {
    path: config.build.assetsRoot,
    filename: '[name].js',
    publicPath: assetsPublicPath
  },
}

```

公共包抽取

```
// webpack.prod.conf.js
const webpackConfig = merge(baseWebpackConfig, {
  new webpack.optimize.CommonsChunkPlugin({
      name: 'base.min',
      minChunks (module) {
        // any required modules inside node_modules are extracted to vendor
        return (
          module.resource &&
            /\.js$/.test(module.resource) &&
            (module.resource.indexOf(
              path.join(__dirname, '../fe/js/lib')
            ) === 0
          )
        ) ||  (
          module.resource &&
          /\.js$/.test(module.resource) &&
          module.resource.indexOf(
            path.join(__dirname, '../node_modules')
          ) === 0
        )
      }
    })
})

```

package.json

```
{
  "scripts": {
    "dev": "webpack-dev-server --inline --progress --config build/webpack.dev.conf.js",
    "start": "npm run dev",
    "lint": "eslint --ext .js,.vue src",
    "test": "node build/build.js test",
    "build:pre": "node build/build.js pre",
    "build": "node build/build.js"
  }
}
```

build.js
```
// 根据npm script命令中传入的参数给NODE_ENV赋值不同的字段
var minimist = require('minimist');
var args = minimist(process.argv.slice(2));
if (args._ && args._.length > 0) {
  process.env.NODE_ENV = args._[0]
} else {
  process.env.NODE_ENV = 'production'
}
```

### 总结
1. 本次重构js库的大小从500kb缩减到310kb左右
2. 一些大型库改为依赖加载(element-ui、lodash)
3. 一些引用但是没有使用到的库删除（moment）
4. 开启eslint以后格式统一，修改了部分语法错误
5. Vue库原来的引入方式，当前环境是‘production’，这就导致一些vue语法错误不会爆出来，但是改构建架构以后当前环境是根据你执行命令改变的，这就导致以前一些vue语法错误，现在统一爆出来，已经修复