---
layout: post
title:  "Android如何自定义一个view——绘制篇"
date:   2015-04-26 16:57:15 +0800
categories: development
tags: android
---

Android中 View的绘制分为三步。

<!-- more -->

1. measure —— 用于得知（子）View的大小
2. layout —— 摆放好（子）View的位置
3. draw —— 真正绘制View的内容

因为Android的layout系统是一个考虑好`相对布局`的一个系统，我们知道`ViewGroup`是继承于`View`的，思想上可以把`ViewGroup`当成是一个`View的组合`

我们看看在三个函数里分别做了什么。

## onMeasure

这个函数主要传入两个参数

```java
void onMeasure(int widthMeasureSpec, int heightMeasureSpec)
```

一个代表宽度的参数，一个代表高度的参数。
这里的宽度参数是父容器的一些参数，它并不仅仅是数值，它用了位运算，根据相应的掩码能得到父容器能给与子容器的宽度，有`EXACTLY`,`UNSPECIFIED`和`AT_MOST`三个值，分别说明：

1. `EXACTLY` 父容器希望子视图有它指定的大小
2. `UNSPECIFIED` 父容器可以无限容纳子容器，子视图要多大都可以
3. `AT_MOST` 父容器指定了最大的大小，让子视图自行决定大小。

根据这些算出大小后，父容器就知道自己应该占多少的空间，同时报告给它的父容器，在这个时候，也可以把子容器应该有的大小记下来，一会在`onLayout`中用。

## onLayout
这个函数声明如下：
```java
void onLayout(boolean changed, int l, int t, int r, int b)
```

第一个参数表明大小位置是否变动过，剩下的4个参数分别代表该容器的`left`，`top`，`right`和`bottom`，容器可以根据这个参数，直接得出它现在的宽度，高度，位置等，如果它是一个`ViewGroup`，那么它可以根据这些参数为它的子视图进行布局。

## onDraw

好了，这个是最后的一个步骤了，就是画。
传入的参数就是`Canvas` 一个画布，你可以在这个画布上绘制你要的各种样式，
这时候调用`getWidth`和`getHeight`都是安全的，因为已经经过了`onLayout`的步骤了。

以上是Android中自定义View最重要的三个步骤，理解了这三步，就在准确的位置，准确的大小画出你想要的图形了。