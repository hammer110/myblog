---
title: 关于移动端js倒计时
date: 2018-05-23 16:24:15
---



#### 问题场景
倒计时是大家经常遇到的需求，但是在ios移动端，当程序运行到后台的时候，会导致倒计时停止，当程序重新打开的时候才会继续。由于最近有几个小伙伴请教这个问题的解决方案，特整理如下

<!-- more -->

#### 解决方案

##### 1. worker和postmessage

利用Worker和postMessage来解决，[具体解决方案](http://www.haorooms.com/post/js_Worker)
由于这个利用最新的api，会有兼容性问题，所以不展开说

##### 2. 利用固定差值计算
此解决方案，是16年在玖富理财端做商城业务的时候想到的解决方案

解决思路：

1. 初始化页面保存两个字段为当前时间戳，startTime和endTime，当然有可能时间戳是后台返回的，那么把后台返回的时间戳赋值给endTime
   
   ```
   // time为倒计时的时间
var time = 1000000
   var endTime = starTime = new Date().getTime()
   ```
   
2. 取差值 
      
   ```
   var diffValue = endTime - starTime （此时为0）
   ```
   
3. 开启倒计时（重点）
   
   这里的倒计时和普通的倒计时的区别在于，每次进入倒计时回调函数的时候，都需要重新获取当前时间戳，然后和endTime做差（curDiffTime），curDiffTime和diffValue继续做差，如果这个差值(mxTime)在正负300之内，那么可以进入下一步，否则的话，会重新修正当前倒计时的时间，endTime加上mxTime，time会减去mxTime，修正结束完成以后，重新计算时分秒渲染页面
需要注意的是endTime也需要随着倒计时每一次加1s或者1000ms   
   
   ```
   var interValId = setInterval(function ( ) {
       if (time <= 1) {
         clearInterval(interValId)
       }
       endTime +=1000
       var starTime2 = new Date().getTime()
       var curDiffTime = (endTime - starTime2)
       if ((curDiffTime>= (diffValue - 300)) &&  (curDiffTime <= (diffValue + 300)) ) {
        time-=1
        renderHourMints(time)
       } else {
         var curtime = Math.abs(endTime- starTime2)
        endTime = endTime + curtime + 1000
        time = time - (curtime/1000) - 1
        renderHourMints(time)
       }
           
   }, 1000)
   ```

#### 总结
这个问题利用的是，差值固定，初始化的时候保存了一个差值diffValue，按照正常流程来说每一次倒计时new出来的时间戳和endTime（endTime每次循环会做累加操作）的差和diffValue的差值会在正负300ms之内。但是如果我中间把程序退出到后台的话，倒计时停止，我重新打开的时候，新的curDiffTime和diffValue的差值就会大于300ms(正常情况这个差值为你退出app到后台的这个时间)，这个时候就会进行时间修正，endTime和time两个字段重新赋值。拿到修正过的值，重新计算，渲染页面


#### tips
上述代码可能有误差，这个文章主要是提供一个思路（请大家忽略蹩脚的命名方式）