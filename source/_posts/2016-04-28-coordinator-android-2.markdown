---
layout: post
title:  "Android 优化交互 —— CoordinatorLayout 与 Behavior"
date:   2016-04-28 18:00:00 +0800
categories: development
tags: android
---

如果你已经很时髦的用上了`AppBar`，`TabLayout`，`FloatActionButton`，以及`Snackbar`的话，我想你多多少少肯定知道`CoordinatorLayout`这个东西。 它的神秘感来自于在布局文件 (xml) 和代码调用上完全看不出和其他组件任何的耦合，却能做出一些神奇酷炫的交互效果。
<!-- more -->


![clipboard.png](https://segmentfault.com/img/bVvfbT)



对，没错，今天我们就着重讲一下`CoordinatorLayout` 是如何工作的，或者说，是如何让别的组件亲密无间的“合作”起来的（以及，顺便会打下一点点的广告）。

## CoordinatorLayout 到底是干嘛的？

顾名思义，`CoordinatorLayout`专注于把它的子View连接起来，使他们之间相互很好的配合。

那么既然是合作，`CoordinatorLayout`的职责充当了一个第三方的角色，通知各个子View之间状态的变换，的确，它也只干了这么一件事，非常纯洁&纯粹。

那么既然有通知，一定需要媒介，总不能把子View全部改造成适合现在这种模式的模样吧？这样也太不OO了，这里的介质就是`Behavior`，`Behavior`是`CoordinatorLayout`用来和各个子`View`通信用的代理类。


## 来自Behavior的代理 —— 布局

![CoordinatorLayout通过Behavior控制子类](https://segmentfault.com/img/bVvesR)

注意箭头的走向，`CoordinatorLayout`是通过`Behavior`去控制子视图，也就代表`Behavior`的数据传导基本上是单向的。当`CoordinatorLayout`需要进行`measure`，`layout`的时候，都会通过`behavior`询问子视图，是否需要进行相应的操作，如果不需要，就进行默认的行为，我们来看下`onLayoutChild`和`onMeasureChild`两个再熟悉不过的行为。

![clipboard.png](https://segmentfault.com/img/bVvets)


![clipboard.png](https://segmentfault.com/img/bVvetu)

可以看`@return`的说明，如果`Behavior`处理了相关的操作，那么就会覆盖`CoordinatorLayout`默认的行为（其实它的默认行为和FrameLayout简直一模一样）

这一节简单的说了`CoordinatorLayout`如何通过`Behavior`来控制子View的布局相关的行为，接下来我们看看重点的`交互`部分。

## 来自Behavior的代理 —— 触摸事件之普通流程

`CoordinatorLayout`的功能当然不仅仅是通过`Behavior`来控制子视图的布局，控制触摸的流程才是大头。

首先我们知道，控制触摸事件，一般有2个：
1. onInterceptTouchEvent.
2. onTouchEvent

这里不解释他们之间的区别，我们看到在`Behavior`中也有这么两个方法。

![clipboard.png](https://segmentfault.com/img/bVveuk)

![clipboard.png](https://segmentfault.com/img/bVveul)

如果你的View所拥有的`Behavior` 处理了相关的事件，那么接下去发到`CoordinatorLayout`上的触摸事件就会像正常流程一样发到这个`Behavior`中。

> 我们终于可以实现在不子类化`View`的情况下，重写它的触摸事件啦。

## 来自Behavior的代理 —— 触摸事件之 NestedScrolling

这才是重点中的重点啊！！
首先，我们来睁大眼睛看！

![clipboard.png](https://segmentfault.com/img/bVveuG)

好，可以看见`CoordinatorLayout`是实现了`NestedScrollingParent`接口的，也就是说，要用到这个特性的话，默认不实现`NestedScrollingChild`接口 (pre Lollipop) 且不调用`dispatchNestedScroll`相关接口的`View`靠一边去！ 【`ListView` 哭晕在厕所】

> `CoordinatorLayout`正是从`NestedScrollingParent`相关的接口中，获取到嵌套滚动相关的参数，再通过`Behavior`传到各个子`View`中，包含`Behavior`的`View`这时候才成为真正处理嵌套滚动的对象，消费掉一些滚动参数后，再把消费掉的数值传回到发生触摸事件的`View`中，达到交互的目的。

![clipboard.png](https://segmentfault.com/img/bVve8e)

`consumed`这个数组可以在`View0`中获取到，表示的意思是它的`NestedScrollParent`消费了多少的滑动量，意味着它能使用的滑动量要减去数组里的值。
这样产生滑动的`View`就通过`CoordinatorLayout`和 其他的`View`的`Behavior` 产生了交互，我们可以在`Behavior` 中给`View`生成一些位置的偏移量，达到视图上移动的效果。

## 来自Behavior的代理 —— layoutDependency
布局依赖，这也是个很重要的东西，比如`FAB`的位置需要在`Snackbar`上边，需要依赖它来操作。
主要有2个接口（用的比较少的暂时忽略）：

1. layoutDependsOn
2. onLayoutDependencyChanged

第二个接口通常在`onPreDraw`或者`onNestedScroll`系列的回调中 最后进行调用，**注意：它并不是在`onLayout`过程中回调的**

第一个很显然是告诉`CoordinatorLayout`，一个View是否依赖于另一个View。

第二个是`CoordinatorLayout`发现存在依赖的时候，把`被依赖方`回调给`依赖方`，**因为这时候，`layout`已经完成，我们可以获取`被依赖方`的所有布局信息，根据布局信息，使用`offset`来决定依赖方的一些位置；**同时在这个时候，你也可以调用`requestLayout`进行重新布局。

## 最后的最后
好了， Guang Gao Time:

> SegmentFault for Android 新版在奋力开发中，带着对`Material Design`的执着，强势归来！

![图片描述][1]

> 欢迎关注我[Github](https://github.com/geminiwen) 以及 [@Gemini](http://weibo.com/coffeesherk/home?leftnav=1) 


  [1]: /img/bVvfr4