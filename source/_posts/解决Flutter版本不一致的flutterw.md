---
title: 解决Flutter版本不一致的flutterw
date: 2019-06-21 19:47:20
categories: Flutter
tags:
---

### 参考

[Flutter 混合开发组件化与工程化架构](https://mp.weixin.qq.com/s/NK0RMuXM2_AJmAbnnvv9SA)

### 前言
开发 Flutter 应用的同学都知道，有个痛点就是如果是团队协作开发的话，就会存在使用的 Flutter 版本不一致的问题，就算只是个人开发，如果需要用 ci 打包的话，打包机上的版本也需要去保持一致，比如 A 同学在开发时，发现 Flutter 低版本有个 bug，升级到高版本就可以解决，但 B 同学并没有同步升级，这就导致在两方打出来的包不一样，如果这个 bug 不明显，那这里就会有很大的隐患，使用 ci 同理。

### gradlew

如果有使用 gradle 的同学会知道，在使用 gradle 构建应用时，会推荐使用 gradlew，这样会使用在 gradle-wrapper.properties 中配置的 gradle 版本，这就是解决了 gradle 版本差异的问题。

### flutterw

同理，我们也可以仿 gradle 这种做法写一个 flutterw 的脚本，我们用 flutterw 来代替 flutter 命令。
首先我们先在 wrapper 目录下创建一个 flutter-wrapper.properties
```
distributionUrl=https://github.com/flutter/flutter.git
flutterVersion=1.6.6
flutterChannel=dev
androidDir=.android
iosDir=.ios
```
上面三个不用解释，androidDir 和 iosDir 是用于指定在哪里生成 `flutter.sdk` 和 `FLUTTER_ROOT` 的配置，将 sdk 目录指向我们的固定版本的 Flutter SDK。接下来我们就可以使用 flutterw 来代替 flutter。

![flutterw](https://user-gold-cdn.xitu.io/2019/6/21/16b77d5a2508de18?w=1212&h=470&f=png&s=859752)

### 原理

flutterw 的实现并不麻烦，首先我们从 flutter-wrapper.properties 读取配置，判断是否需要下载 Flutter SDK，我们会将 SDK 下载在 .flutter 目录下，接着判断是否需要切换版本，然后将我们的参数传递给 flutter 脚本，最后设置 SDK 目录。

### 源码

[github](https://github.com/LinXiaoTao/flutterw)

这里我只实现了 sh 脚本，bat 版本后面有空再实现了，或者哪位大佬实现下，提个 PR，感激不尽。