---
title: 仿即刻Flutter版本
date: 2019-08-07 19:48:22
categories: Flutter
tags:
---

### 题外话

即刻是我个人用的比较频繁的 APP 之一，但在我的手机一直有个 bug，播放视频时总是画面没有变化，需要手动推动下进度条才行，在更新了几个版本也没解决这个问题。再加上刚学了 Flutter，就尝试做了个 Flutter 版本的即刻，即刻页面比较多，所以没有完全开发完，只开发了首页的信息流，信息详情，视频播放等几个用的最多的页面。

> 接口用的是即刻的接口和图标，**这里是为了学习，请勿用做其他用途，一切后果由自己承担，版权归即刻所有**


因为我本身学习 Flutter 的时间也才几个月，所以代码质量还是有所欠缺的。

最近即刻被暂停服务了，所以 API 也调用不了，只能看看之前录制的片段。

源码地址：[today](https://github.com/LinXiaoTao/today)

### 效果

国际惯例，先实现的效果图：

![jike](https://user-gold-cdn.xitu.io/2019/8/7/16c6aa0d7a121ba0?w=320&h=569&f=gif&s=8296263)

这里面属首页的信息流最复杂，因为涉及了很多的样式，有几个花了时间实现的：

- @ 人的效果，文字中有多个高亮显示的标签，点击可以跳转页面
- 搜索栏和列表的嵌套滚动

第二个花时间的是个人详情页面中，整体页面的嵌套滚动。

### 实现

首先看下项目的整体结构，这里面代码整体重构过一次，之前使用 ScopedModel，后面觉得不太灵活，又换了相对热门的 Bloc 架构。

#### bloc

这个目录存放整个应用所有的 bloc 文件，按功能模块划分，最外层有一个 `blocs` 用于导出所有的 bloc 文件，每个模块中有四个文件，分别是：

- bloc.dart：导出当前模块的 bloc
- xxx_event.dart：bloc 中的 event
- xxx_state.dart：bloc 中的 state
- xxx_bloc.dart：处理逻辑

#### network

网络层的代码在 network 目录下，用的是 [dio](https://github.com/flutterchina/dio) 框架，这个框架用起来比较方便，API 的封装有点类似 okhttp，这里也写了个拦截器 `BusinessInterceptor` 用于处理 token 信息。接口定义放在 `ApiRequest` 里面，相关的数据模型放在 `model` 目录下，用的是 [**json_serializable**](https://github.com/dart-lang/json_serializable/tree/master/json_annotation)。

#### UI

所有的页面存放在 ui 目录下，按功能模块划分，其中 `ui_base.dart` 存放一些基类，颜色常量等等。

#### flutterw

关于 Flutter 的版本问题，估计很多小伙伴也挺头疼的，特别是需要协作开发的。这里我自己写了一个类似 gradlew 的 flutterw，使用特定的版本的 Flutter 去编译，只需要将 flutter 改用为 flutterw 即可。

> 关于 flutterw 的更多介绍：[解决Flutter版本不一致的flutterw](https://juejin.im/post/5d0c39326fb9a07ef63fe4b0)

### 最后

模仿的页面不多，但还是花了不少时间去写的，talk is cheap show me the code。