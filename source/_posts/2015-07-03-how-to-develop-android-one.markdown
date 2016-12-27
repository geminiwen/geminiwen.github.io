---
layout: post
title:  "#土豆记事#教你开发Android App之 —— 认识Android开发工具"
date:   2015-07-03 18:00:00 +0800
categories: development
tags: android
---

注： 这是为想入门Android的新手准备的一篇文章

<!-- more -->

想学习写Android App么？ 其实很简单，哦，再简单之前，也要先学`java`。

.......

好了，你入门`java`了，那就可以来看看用一天时间写一个App是多么容易。
我们来写个`记事本`吧。

OK，先下载`Android Studio`，`Android Studio`是Google官方推荐的IDE，能快速的开发Android App.

下载地址 http://developer.android.com/sdk/index.html

## 第一步创建工程
我们新建一个项目，点`Start a new Android Studio project` 即可。

![clipboard.png](https://segmentfault.com/img/bVmzp4)
----

## 第二步 输入一些项目的属性。
`Application name`就是你应用的名称，`Company Domain`是你公司的域名，如果你是个人开发者，就写你自己的域名即可。
`Package name`是包名，理论上要求**在地球范围内唯一**，它标识了你这个App。
![clipboard.png](https://segmentfault.com/img/bVmzp5)
---

## 第三步 输入支持的最低版本的SDK
这里我们只创建手机和平板应用。`Minimum SDK`是什么呢？比如你如果选择Android 5.0，那么5.0以下的设备就不能安装你的App，兼容性越差。 我们知道Android每次更新版本都会增加许多新特性，也就意味着Android SDK版本越高，特性越多，兼容性越差。 如果你的App受众很大，建议选择低版本的SDK，并采用一些替代方案来实现你要的功能，我们这里如果只是为了学习SDK，可以直接使用`Lollipop`，否则我推荐`IceScreamSandwich`。
![clipboard.png](https://segmentfault.com/img/bVmzqy)

## 第四步 创建一个页面

这里要介绍下，Android中的UI都是通过`Activity`来呈现，一个Activity可以理解成你看到的“窗口”，你从一个列表跳到一个详情，可以简单的认为从一个`Activity`跳到另外一个`Activity`。这里`Android Studio`会通过向导模式帮你创建一个入口`Activity`，我们可以选择不创建，或者第一个`空白Activity`（`Blank Activity`）
![clipboard.png](https://segmentfault.com/img/bVmzqP)

## 第五步 设置Activity的属性
这一步可以设置`Activity`的java类名，`Activity`的布局，`Activity`显示的标题以及菜单的资源。
`Activity`通过布局文件（`layout`）来决定显示怎么样的界面，`Title`是Activity顶部显示的一行文本，`menu`是Android手机点menu键或者点右上角的`...`显示的菜单。

![clipboard.png](https://segmentfault.com/img/bVmzqV)

## 完成创建
点击`finish`，一个工程就创建完了，这就是最基本的创建工程的步骤。


> 本文提到的项目源码地址：https://github.com/geminiwen/tudounotepad
> 欢迎留言[Github](https://github.com/geminiwen)或者[@geminiwen](http://weibo.com/coffeesherk/home?wvr=5)