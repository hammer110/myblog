
---
title: vue-cli升级webpack4
date: 2018-09-12 21:39:15
---

mark一下vue-cli升级webpack4的整个过程

 <!-- more -->
 
### 更新库

```
npm install
  webpack@4
  webpack-dev-server@3
  webpack-cli
  eslint-loader@2
  vue-loader@15
  html-webpack-plugin@next
--save-dev 

```

### 修改vue-cli配置

#### webpack.base.conf.js

```
// 头部引入该文件
const vueLoaderPlugin = require('vue-loader/lib/plugin')

module.exports = {
  plugins: [
    new vueLoaderPlugin()
  ]
}
```
#### webpack.dev.conf.js

```
const devWebpackConfig = merge(baseWebpackConfig, {
  mode: 'development'
})
```

#### webpack.prod.conf.js

```
const webpackConfig = merge(baseWebpackConfig, {
// 指定当前运行环境方式改变，原来是通过webpack.DefinePlugin，现在改为mode
   mode: 'production',
   plugins: [
     // contenthash改为chunkhash
     new ExtractTextPlugin({
          filename: utils.assetsPath('css/[name].[chunkhash:5].css'),
          allChunks: true,
     })
   ],
   // 注释掉所有optimize.CommonsChunkPlugin相关
   代码
   
   splitChunks: {
      cacheGroups: {
        "base.min": {
          test: /[\\/]fe[\\/]js[\\/]lib[\\/]/,
          chunks: 'initial',
          name: 'base.min'
        },
        vendors: {
          test: /[\\/]node_modules[\\/]/,
          chunks: 'initial',
          name: 'vendors',
        }
      }
    },
    runtimeChunk: { name: 'manifest' }
  }
})

```

### 多页面打包单独配置(单页面不需要配置此处)

html-webpack-plugin chunksSortMode修改为none


### 总结

1. 个人感觉optimization splitChunks是这次改变最大的一点，原来的架构配置vendor以后再配置base.min的提取，总是提取失败。最后不得不把base.min和vendor.js打包在一起，但是base.min.js会经常改动，就导致vendor的hash值总是改变，浪费性能，升级最新版本以后可以同时提取出来base.min和vendor
2. 4.0也会提取公共css啦，多个页面公用的css也会提取到vendor.css里面，更加方便缓存

