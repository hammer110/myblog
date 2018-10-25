---
title: 类数组转数组的几种方式
date: 2018-07-04 19:27:15
---

### 什么是ArrayLike(类数组)

1. 拥有length属性，其它属性（索引）为非负整数(对象中的索引会被当做字符串来处理，这里你可以当做是个非负整数串来理解)
2. 不具有数组所具有的方法
3. function内的anguments和DOM方法的返回结果
<!-- more -->

### 转换方法

#### 1. slice

```
// slice
function changeType() {
 return Array.prototype.slice.call(arguments)
}
Array.isArray(changeType(1,2,3,4,5))  // true
```
slice这个方法有点小插曲，刚开始看[w3school](http://www.w3school.com.cn/jsref/jsref_slice_array.asp)的文档显示slice接受2个参数，并且第一个是必填，但是我们这里调用的时候没有传入任何参数，还能调用成功。后来去[mdn](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/slice)上查看，发现slice第一个参数不是必填，如果不传入的话默认为0，所以看文档还得去mdn比较靠谱

#### 2. splice

```
// splice
function changeType() {
 return Array.prototype.splice.apply(arguments, 0, arguments.length)
}
Array.isArray(changeType(1,2,3,4,5))  // true
```

#### 3. concat

```
// concat
function changeType() {
 return Array.prototype.concat.apply([], arguments)
}
Array.isArray(changeType(1,2,3,4,5))  // true
```

#### 4. from

```
// Array.from
function changeType() {
 return return Array.from(arguments)
}
Array.isArray(changeType(1,2,3,4,5))  // true
```
#### 5. new Array

```
// 类数组转换
function changeType() {
 return new Array(...arguments)
}
Array.isArray(changeType(1,2,3,4,5))  //true
```

#### 6. for循环

```
// for循环
function changeType() {
 var arr = []
 for (var i = 0; i < arguments.length; i++) {
     arr.push(arguments[i]) 
 }
 return arr
}
Array.isArray(changeType(1,2,3,4,5))  // true
```

## 总结

其实转换的方法应该还有很多种，但是个人感觉大致分为3种
1. new array
2. 数组方法调用完以后会返回一个新的数组（splice、slice）
3. 循环for等

其实网上大部分都是基于slice来做的，今天正好和一个朋友讨论这个话题的时候想到了其他的一些，特此写一篇文章记录一下


