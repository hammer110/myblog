---
title: webpack 图片单独部署cdn
date: 2018-09-15 21:08:15
---

由于最近想实现图片和js、css cdn服务器的分离，实现图片单独在一个服务器，这样可以突破浏览器并发数的限制，实现首屏渲染更快，以下是这次拆分过程的简单记录

 <!-- more -->

## 1. __webpack_public_path__

首先想到的就是在入口文件顶部配置__webpack_public_path__，但是在实践过程中发现不满足实际需求，因为我们在开发中会编写一些异步组件，如果按照这个方式那么异步组件的js域名也会变成改完以后的，这样就不能做到只图片资源单独修改

### 异步组件的加载方式

啰嗦一下为什么__webpack_public_path__不行

```
import('./app.vue').then(data => {})
```
上述代码是我们在开发过程中经常会编写的，但是这段代码等我们build完成以后在浏览器内访问时是怎么一个加载过程呢？

这就需要先简单说一下manifest.js，这个js封装了一个webpackjsonp对象，比如我加载app.vue这个组件的时候，它会先去manifest.js对照表内找到这个js编译完成以后对应的hash值，比如

```
 'app': "7464371616158c3f7e36"
```
接着拼接这个app.js对应的src链接

```
a.src = o.p + "static/js/" + e + "." + ".js"
```
o.p就是指向了build/webpack.base.conf.js的publicPath，e代表的是上文中的hash，这样就拼接成了一个完整的js地址，然后通过动态创建script标签append到html中，这样就完成了一个js的异步加载

***回到上一个话题，当我们修改__webpack_public_path__的时候会同步把manifest.js内的o.p覆盖掉，导致异步js加载的域名同步修改，但是这不是我们最终期望的效果***

## 2. nodejs脚本统一替换
发现上面这条路径不行的话，我突然想到不如写一个node脚本统一替换css内的图片地址，脚本比较简单如下
```
fs.readFile(file, 'utf-8', function (err, data) {
    if (err) {
      console.log(file, '替换url失败')
      process.exit()
    }
    const replaceHostData = data.replace(replaceHost.regex, replaceHost.curHost)
    fs.writeFile(file, replaceHostData, {encoding: 'utf-8'}, function (err, data) {
      if (err) {
        console.log('写入文件失败')
        process.exit()
      }
      var arr = file.split('/')
      console.log(arr[arr.length - 1], '替换成功')
    })
})
```
过程比较简单，还是按照原有方式build，但是build完成以后通过这段脚本替换掉css内所有的图片cdn地址

这种方式的确没太多问题，但是只适用于在css内引入的图片，如果在js内引入的图片就不行了

```
// index.js
var image = require('./image/test.jpg')
```
这种方式引入图片最终会编译成

```
o.p + 'static/'+ 'image/test.jpg'
```
由于js内的cdn地址都是o.p这种变量，导致nodejs脚本跑的时候会匹配不到

## 3. file-loader

file-loader是对一些多媒体资源路径进行处理的，它提供了一个配置参数publicPath，可以配置图片的cdn地址

vue-cli中用的是url-loader，不过不用担心，它只不过对file-loader做了一层封装，你配置的option最终都会传入到file-loader内

```
{
    test: /\.(png|jpe?g|gif|svg)(\?.*)?$/,
    loader: 'url-loader',
    options: {
      limit: 1,
      name: utils.assetsPath('img/[name].[hash:7].[ext]'),
      publicPath: "http://qq.com"
    }
  },
```
这样配置完成以后所有的图片资源都会变成http://qq.com/static/image/test.jpg  ***记住不是o.p + imageUrl的方式***，是直接写死了图片链接。这样同时也代表，如果我在file-loader内配置了publicpath，后期图片的链接不能通过__webpack_public_path__动态改变

走到这里大家肯定以为这个方法是万全之策了，但是实际上还是有点小瑕疵，那就是项目根目录的static这个文件夹，因为放到这个目录里面所有的资源都不会走webpack编译，导致在file-loader内配置的publicPath、
webpack.base.conf.js内配置的publickPath对static目录下的资源无效。不过好在我们这个项目static下面没有放过静态资源

***
tips: 如果css内引入static目录下的图片，那么这个图片加载的域名会和css的域名一样走cdn，如果是非css，那么就会和html的域名一样
***


## 总结

上文是我这次做image单独部署cdn服务器的一些实践步骤，简单make一下