---
layout: post
title:  "Android 程序员学习 iOS ——UIViewController 和 Layout System"
date:   2016-01-07 18:00:00 +0800
categories: development
tags:
	- android
	- ios
---

Hello，通过[Android程序员 如何入门iOS ——故事从这里开始](http://segmentfault.com/a/1190000004268513) 作为一个Androider 去看iOS程序的目录结构应该算有个大概的理解了，接下去我们小小介绍下和我们交道打的最多的`UIViewController`。 
<!-- more -->

## 什么是ViewController
Android 开发者们不会对`activity`有陌生的感觉吧？那么这里的`UIViewController`就可以理解成`Android`中的`activity`，`UIViewController`有一个不同的地方, 就是它和某一个`view`是强耦合的，在逻辑上，一个`UIViewController` 必然带一个`view`（其实不带`view`的`activity`好像也没什么价值= =）。

## iOS Layout System 和 Android Layout System
我们这里需要清楚明白一点的是，`iOS`不像`Android`，带了一个`layout system`，它在不采用`autolayout`的情况下并不会自动布局，`Android`的一个`ViewGroup`的生命周期经历3个阶段，分别是
> 1. measure
> 2. layout
> 3. draw

`Android`在大部分情况下，`ViewGroup`都会自动的为它的`子view`撑开足够的空间，用以正确显示`View`。这么智能的做法是在`measure`和`layout`中完成的。

`iOS`的绘图性能一直完爆`Android`的其中一个原因就是因为它简单的布局系统不会因为布局的复杂性增强而增加计算量。

如果不采用`autolayout`，那么在`iOS`中，所有的`View`有个初始化方法叫做`initWithFrame:` 传入一个`CGRect`矩形对象，矩形定义了 (`x`,`y`,`width`,`height`)，这四元 不就是我们帮系统完成了一次`measure`和`layout`么？ 那么`iOS`只用尽管`draw`就行了。

## iOS UIViewController LifeCycle
这里不提`Activity LifeCycle`的原因是，`Activity`的`LifeCycle`好像和`View`一点关系都没有
> `onCreate` - `onStart` - `onResume` - `onPause` - `onStop` - `onDestory` 
这些和`Activity`本身有关，似乎在哪都找不到`View`相关的事件回调，但是在`iOS`的`UIViewController`中，却有
> `viewWillAppear` - `viewDidAppear` - `viewWillDisappear` - `viewDidDisappear` 
好像每一个都和`View`有关，不愧名字为`ViewController`啊~

我们从`Android`迁移到`iOS`首先写`UIViewController`的时候，一个后遗症就是会去找`ViewController`的生命周期，其实不用想太多，因为iOS充分利用了`getter`和`setter`的便利性，在我们对`UIViewController.view`的访问过程中，会调用`loadView`和`viewDidLoad`这2个回调方法，因此，我们原先在`Activity`中, `setContentView`之后干的事情，就可以在`viewDidLoad`中去做了，至于`ViewController`是否显示消失，那么就在其它四个回调中去做我们想要做的事。

参考官方文档：https://developer.apple.com/library/prerelease/tvos/documentation/UIKit/Reference/UIViewController_Class/

## 总结
本文我们简单介绍了`UIViewController`和`Activity`自身`生命周期`的不同和两个系统`布局系统`的不同，希望对大家有所帮助，由于我自己也刚刚入门iOS，写的文章可能漏洞较多，欢迎大家补充。

当然学习建议还是  **多看官方文档**

