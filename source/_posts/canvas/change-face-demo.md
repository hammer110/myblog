---
title: 换脸纯前端解决方案
date: 2018-07-15 15:15:15
---

以前一个朋友做过类似的需求，一直比较好奇是怎么实现的，周末闲来无事搞了一个小demo，[demo地址](http://demo.sinker.club/change-face/)，本demo只截取了关键部分代码
<!-- more -->

### 人脸识别

人脸识别是整个环节最关键的一步，只有识别出来图片上人脸的位置和大小才能进行下一步，我们是基于[trackingjs](https://trackingjs.com/)来实现的

```
<script src="http://demo.sinker.club/change-face/tracking-min.js"></script>
<script src="http://demo.sinker.club/change-face/face-min.js"></script>
<script src="https://unpkg.com/spritejs/dist/spritejs.min.js"></script>

```
需要注意的是单纯的引入[trackingjs](https://trackingjs.com/)是不可以的，还需要[face.js](http://demo.sinker.club/change-face/face-min.js)库配合

```
let rect = {}
// 此处不仅可以配置face还可以配置eye、mouth，不过都需要在头部引入对应的js库
const tracker = new 
tracking.ObjectTracker(['face']);
// #myImg 是在html中有一个img标签，用于回显人物图片
tracking.track('#myImg', tracker);
// 添加监听事件，当计算出来图片中人脸的位置会触发该回调函数
tracker.on('track', function (event) {
    if (event.data.length === 0) {
        alert("无人脸")
    } else {
    // 由于本demo只有一张图片，如果是多张图片，返回来的是每张图片人脸的位置
      rect = event.data[0]
    }
})
```
### 换脸

换脸本质上是基于canvas来做的，首先把人物图片当作背景绘制到canvas画布上，通过tracker得到人脸的位置，然后当用户继续上传人脸icon的时候，会把想要替换的人脸icon绘制到该画布指定的位置上实现换脸，canvas我们是基于[spritejs](http://spritejs.org/#/zh-cn/guide/)来实现的

```
// 监听用户上传图片的change事件
inputImage.addEventListener('change', function(e) {
    const freader = new FileReader(); 
    freader.readAsDataURL(e.target.files[0]);  
    freader.onload = function(event) {
    // 在img标签内回显图片，让tracker进行计算
        myImg.src = event.target.result
        // 把该人物图片绘制到canvas画布上
        addCanvasBg(event.target.result)
    } 
}, false)
```

绘制人物图片

```
// addCanvasBg
const {Scene, Sprite} = spritejs;
const paper = new Scene('#container', 200, 300)
const layer = paper.layer()
const sprite = new Sprite(url)
sprite.attr({
    bgcolor: '#fff',
    pos: [0, 0],
    size: [200, 300]
})
layer.append(sprite)
```

绘制上传的人脸icon到人物图片指定位置

```
// addCanvasIcon
function addCanvasIcon ({width, height, x, y}, url) {
    let box1 = new Sprite(url)
    box1.attr({
        anchor: 0,
        size: [width, height],
        pos: [x, y],
    })
    layer.append(box1)
}
```

到这里已经把新上传的人脸icon换到指定的人物图片上

### 导出图片

可能有一些需求需要把换完脸的图片存储到服务器，方便用户分享。那么导出图片怎么做的呢？

```
const canvas = document.querySelector('#container canvas')
canvas.toDataURL()
```

导出图片本质上是基于canvas的原生api [toDataURL](https://blog.csdn.net/oulihong123/article/details/73927514)来实现的
扩展一下，其实好多web端实现图片压缩的库也都是基于toDataURL的第二个参数来实现的，比如[lrz](https://github.com/think2011/localResizeIMG)

### 总结
1. 上述解决方案是基于trackingjs、sprite.js纯前端的解决方案，需要注意的是trackingjs在识别一些人物脸部不是特别清晰的图片的时候，他会识别失败
2. 该文章只截取了部分实现代码，建议去[这里](view-source:http://demo.sinker.club/change-face/)查看完整代码

3. 如果想要查看换完脸的图片，请点击导出图片，会在console内打印出来该图片的base64代码
4. 该[demo](http://demo.sinker.club/change-face/)建议在pc端打开


