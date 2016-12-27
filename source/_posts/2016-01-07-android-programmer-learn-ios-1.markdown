---
layout: post
title:  "Android 程序员学习 iOS  ——故事从这里开始"
date:   2016-01-07 18:00:00 +0800
categories: development
tags:
	- android
	- ios
---


最近因为公司的一些原因，需要涉及iOS开发。在坑里摸爬滚打了2周之后，就写个入门心得吧。
在一切的一切开始之前，你要先会`Objective-C`或者`Swift`（喜欢哪个用哪个，你开心就好）。

然后，要准备一台`OS X`的电脑，并安装好`Xcode`，然后一切就可以开始了。

打开`Xcode`，然后新建一个项目，看到的界面是这样的（其实和`Android Studio`的模式很相似啦）

![clipboard.png](https://segmentfault.com/img/bVr4z0)
看看几个项目，你大概也理解了分别要创建怎么样的应用，它提供一个模板，然后可以快速创建出某种结构的程序。大部分情况的应用是属于`Tabbed Application`，也就是带`Tab`的程序。


## 文件概念迁移

创建好一个应用后，我们能看到`AppDelegate`，`storyboard`，`plist`之类的文件。这些分别是啥呢？


*我来个简单干脆的映射，方便理解，但是不精准，如有错误，感谢指出*

|iOS|Android|备注|
|--|--|--|
|`Info.plist`|`AndroidManifest.xml`|记录应用程序的一些元数据|
|`AppDelegate`|`Application`|管理整个`Application`的生命周期|
|`ViewController`|`Activity`|负责管理`View`，管理整个应用程序的交互|
|各类`storyboard`文件|各类`xml`文件|定义一些布局，一些iOS程序员习惯在代码里写布局，可能用不到`storyboard`|


`Android`程序始于`Application`的`onCreate`方法，`iOS`始于`AppDelegate`的`application didFinishLaunchingWithOptions`方法，这里唯一不同的是`Android`的`launch activity`只能使用`AndroidManifest.xml`指定，但是`iOS`可以使用代码去指定。

```
self.window.rootViewController = xxxx;
```

## 从Activity到UIViewController

`Android`中的`Activity`和`View`并没有强制关联，但是`iOS`中的`UIViewController`默认都带一个`View`，你可以把这个认为是`Activity`中`Window`的`decorView`，是所有`View`的父容器。当你生成好一个`UIViewController`之后，你往它的成员变量`view`中添加你的视图即可，如果你是从`storyboard`中生成的`UIViewController`，那你构建的`View`就会自动添加进来。

`Android`一切初始化的地方，我们习惯在`onCreate`中使用`setContentView`之后，然后用`findViewById`获取到控件的实例，为控件绑定一些监听器，而`iOS`中，我们开始的地方，大部分是`UIViewController`的`viewDidLoad`方法，我们使用代码生成我们要的控件，同时为控件绑定事件，或者使用`storyboard`的`Outlet`系统绑定到我们的类成员变量上，并生成事件监听。

所有故事，都是从这里开始对吧？

接下去要做的事就是根据用户和控件的交互，在视图上做出相应的反馈即可。

## 界面之间的跳转

`Android`的跳转使用`Intent`从一个`Activity`跳转到另外一个`Activity`。
而`iOS`中，我们在跳转之前，要做的事情就是生成我们的目标`UIViewController`，然后使用我们想要的方式跳转。 `iOS`为我们提供了几种模式跳转，最常见的有2种，使用`navigationController`和`pushModal`，大部分的`iOS`程序，顶部都有一个导航条，它由一个更高级抽象的`NavigationController`进行控制，就和`Android`中的`Task`概念类似，我们每次`pushViewController`，就会在它的栈中压入一个`ViewController`。而`pushModal`看名字就知道，是弹出一个模态框，它的返回操作一般只有关闭一个按钮，具体使用哪种方式，需要和产品的概念相呼应才行。

UI的相关介绍就到这，接下去有时间，我们谈谈`Android`中的`Handler`在`iOS`中以什么样的方式存在
