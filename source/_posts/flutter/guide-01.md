---
title: flutter集成blog
date: 2018-07-02 11:24:15
---

[flutter](https://flutterchina.club/)是最近比较火的移动ui框架，可以快速在ios和Android上构建原生用户界面(和rn/weex很类似)，最近想做一个app把blog集成进去，所以用的这个解决方案

<!-- more -->

### 搭建环境

#### 1. clone flutter源码

```
git clone -b beta https://github.com/flutter/flutter.git
```

#### 2. 设置环境变量

设置环境变量，把下面的信息配置到~/.bash_profile内，如果你用到了item2 zsh的话，需要把下面的配置信息配置到~/.zshrc内，最后执行 source ~/.bash_profil或者source ~/.zshrc

```
export PUB_HOSTED_URL=https://pub.flutter-io.cn //国内用户需要设置
export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn //国内用户需要设置
export PATH= PATH_TO_FLUTTER_GIT_DIRECTORY/flutter/bin:$PATH
```
***需要注意的是PATH_TO_FLUTTER_GIT_DIRECTORY为克隆Flutter的git repo的本地路径，记得替换过来***

上述配置完成以后可以在命令行里面执行

```
flutter --version
// 我本地的输出信息
Flutter 0.5.6-pre.114 • channel master • git@github.com:flutter/flutter.git
Framework • revision af5d4c6882 (28 hours ago) • 2018-06-30 17:17:35 -0700
Engine • revision 6fe748490d
Tools • Dart 2.0.0-dev.63.0.flutter-4c9689c1d2
```
然后执行一下 

```
flutter doctor
```

### 初始化项目(vscode)

#### 1. 安装插件

flutter、Dart Code

#### 2. 新建项目

view -> command Palette -> 输入flutter ->  new project

#### 3. 启动

1. 找到一个安卓手机开启开发者模式
2. 手机连接上电脑
3. vscode底部有个device选择，选中你的手机
4. Debug -> start debugging
5. 经过漫长的启动过程，你在手机上可以看到你正在开发的app

### webview集成blog

#### 1. 安装第三方库
flutter如果需要用webview需要下载第三发库flutter_webview_plugin

```
// 在pubspec.yaml的dependencies新增flutter_webview_plugin
dependencies:
  flutter:
    sdk: flutter
  # The following adds the Cupertino Icons font to your application.
  # Use with the CupertinoIcons class for iOS style icons.
  cupertino_icons: ^0.1.2
  flutter_webview_plugin: "^0.1.6"
```
配置完成以后安装

```
flutter packages get
```
#### code

```
import 'package:flutter_webview_plugin/flutter_webview_plugin.dart';
String selectedUrl = "https://www.sinker.club";

void main() => runApp(new MyApp());

class MyApp extends StatelessWidget {
  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      title: 'sinker.club',
      theme: new ThemeData(
        primarySwatch: Colors.blue,
      ),
      routes: {
        "/": (_) => new WebviewScaffold(
              url: selectedUrl,
              appBar: new AppBar(
                title: new Text("sinker.club"),
              ),
              withZoom: true,
              withLocalStorage: true,
            )
      },
    );
  }
}
```

### 构建/发布

```
// 打包好的发布APK位于<app dir>\/build/app\/outputs\/apk\/app-release.apk
flutter build apk  // 直接打出来apk的包

flutter install // 直接安装到调试手机上
```

### 总结

其实上述过程在官网都有，主要是在环境搭建那一块花的时间比较多一点，以后慢慢吧blog慢慢转成flutter

[sinker.club](https://www.sinker.club/app/sinker-release.apk
) apk体验地址


