---
title: 基于fabric.js实现图片可复制
date: 2018-10-14 22:11:15
---

> 需求：当用户按住这个图片的时候，移动鼠标可以clone出来一个新的图片。鼠标移动，这个clone出来的图片跟着移动，源图还在初始化的位置，不跟随鼠标移动

<!-- more -->

### 需求拆解

1. lock源图位置，不能让它可以被拖拽
2. clone出来一个新的图片，这个clone出来的图片可以被拖动
3. 发送信令同步给其他端，同步创建和绘制
4. 可以被垃圾桶删除

### lock源图位置

lock源图位置第一步想到的就是给这个图片设置属性evented，但是设置以后，图片点击事件不会被触发，遂抛弃此方式。经过查看文档，Object下面有2个api, lockMovementX、lockMovementY，可以锁死该对象不能往x、y轴移动

### clone图片

复制图片相对来说比较简单，直接调用clone这个api即可，但是复制出来后拖动的时候，发现拖不动，后来特意把clone出来的图片，位置放到一倍，才发现，原来复制出来的图片在源图的下面，导致鼠标拖动图片的时候实际拖的源图，由于源图已经被锁死

![](http://fe-static.sinker.club/image/fabric/fabric-clone-1.png)

后来想到能不能提升被clone出来图片的层级，经过查询api，发现有两个api bringForward、bringToFront可以提升图片层级，但是经过尝试都失败，被clone出来的图片还是在最下面。接着又想到一个思路，既然被clone出来的新的图片在最下面，那么直接把上面这个图片（源图）状态标记为可以移动，不可复制。把被clone出来的图片状态标记为可复制，不可移动。

***结局思路就是，由于clone的图片层级在源图下面。那么clone的时候直接把clone出来的图片和源图的状态进行了互换***

![状态图](http://fe-static.sinker.club/image/fabric/WechatIMG82.jpg)

### 其他端同步绘制
同步绘制比较简单，只需要在每个被复制的图片上标记上源图的id，那么其他端根据这个cloneId，找到对应的图片复制出来

![cloneid](http://fe-static.sinker.club/image/fabric/WechatIMG81.jpg)

实际情况会和上图一样，不过这样也会有问题的，应该把所有cloneid都标记为最初始图片的也就是cloneId: 1

### 被垃圾桶删除
这个功能其实是原来就有的，不过后来手残，不小心给删掉一些关键但代码，导致图片和垃圾桶碰撞检测失败

```
canvas.on('object:moving', function () {
   image.setCoords()
})

```

当图片位置改变的时候需要手动调用setCoords 更新该对象的acoords信息，否则后期和垃圾桶做碰撞检测的时候会失败（acoords存储的是图片4个边的位置信息）

***后来get到，不管是画线，还是拖动物体其实都是fabric.js底层做的，它在不同的时机fire你已经定义的对应回调函数而已***

