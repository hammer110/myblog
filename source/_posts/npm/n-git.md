---
title: n-git使用指南
date: 2018-09-07 21:43:15
---

## n-git使用指南

> 此库主要是根据在我司开发过程中遇到的问题，对git指令做了二次封装，方便在开发中使用\

<!-- more -->

### 安装

```
 mac   ： sudo npm intall -g n-git
 window： npm install -g n-git
```

项目根目录创建.ngit文件，配置如下

```
module.exports = {
  "name": "ljj3"
}
```
创建分支的时候他会根据你配置的name字段，自动拼接对应的分支名字

### 常用指令集锦

#### 1. help

```
ngit --help 
打印出来所有可用的命令
```

#### 2. -c

```
ngit -c newbranchName

Switched to a new branch 'ljj3-20180907-newbranchName'
```

### 3. -l

该命令会过滤出来所有包含该关键字的分支名字，如果该命令后面不跟任何参数，会打印出来所有分支名称

```
ngit -l "ljj3"

[ 'ljj2-20180803-galleryZoom',
  'ljj2-20180806-allFail',
  'ljj3-20180808-pptTime',
  'ljj3-20180814-pptqy',
  'ljj3-20180817-fixDoubleImage',
  'ljj3-20180817-fixErrorImage',
  'ljj3-20180823-changeport',
  'ljj3-20180828-fixNotRender',
  'ljj3-20180907--o',
  'ljj3-20180907-testGitBranch' ]
```

### 总结

此npm命令只是闲来无事做着玩的小东西，还有一些对git命令的封装比如-a、-m都没做详细解释，自己尝试吧