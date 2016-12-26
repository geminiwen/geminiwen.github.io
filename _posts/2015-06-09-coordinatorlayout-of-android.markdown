---
layout: post
title:  "Android Support Design 中 CoordinatorLayout 与 Behaviors 初探"
date:   2015-06-09 18:00:00 +0800
categories: android
---

在`Android M Preview`发布后，我们获得了一个新的`support library` —— `Android Design Support Library` 用来实现Google的`Material Design` 提供了一系列符合设计标准的控件。

其中有众多的控件，其中最复杂，功能最强大的就是`CoordinatorLayout`，顾名思义，它是用来组织它的子views之间协作的一个父view。CoordinatorLayout默认情况下可理解是一个`FrameLayout`，它的布局方式默认是一层一层叠上去。
那么，`CoordinatorLayout`的神奇之处就在于`Behavior`对象了。
看下`CoordinatorLayout.Behavior`对象的 Overview
> Interaction behavior plugin for child views of CoordinatorLayout.

> A Behavior implements one or more interactions that a user can take on a child view. These interactions may include drags, swipes, flings, or any other gestures.

可知`Behavior`对象是用来给CoordinatorLayout的子view们进行交互用的。
`Behavior`接口拥有很多个方法，我们拿`AppBarLayout`为例。`AppBarLayout`中有两个`Behavior`，一个是拿来给它自己用的，另一个是拿来给它的兄弟结点用的，我们重点关注下`AppBarLayout.ScrollingViewBehavior`这个类。

我们看下这个类中的以下方法
### 0. dependency
```java
 public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency)     {
    return dependency instanceof AppBarLayout;
}
```

这个方法告诉`CoordinatorLayout`，这个view是依赖`AppBarLayout`的，后续父亲可以利用这个方法，查找到这个child所有依赖的兄弟结点。

### 1. measure
```java
public boolean onMeasureChild(CoordinatorLayout parent, View child, int parentWidthMeasureSpec, int widthUsed, int parentHeightMeasureSpec, int heightUsed)
```
这个是`CoordinatorLayout`在进行`measure`的过程中，利用`Behavior`对象对子view进行大小测量的一个方法。
在这个方法内，我们可以通过`parent.getDependencies(child);`这个方法，获取到这个child依赖的view，然后通过获取这个child依赖的view的大小来决定自身的大小。

### 2. layout
```java
 public boolean onLayoutChild(CoordinatorLayout parent, View child, int layoutDirection)
```
这个方法是用来子view用来布局自身使用，如果依赖其他view，那么系统会首先调用
```java
public boolean onDependentViewChanged(CoordinatorLayout parent, View child, View dependency) 
```
这个方法，可以在这个回调中记录`dependency`的一些位置信息，在`onLayoutChild`中利用保存下来的信息进行计算，然后得到自身的具体位置。

### 3. nested scroll

```java
 public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout, AppBarLayout child, View directTargetChild, View target, int nestedScrollAxes)
```

```java
public void onNestedPreScroll(CoordinatorLayout coordinatorLayout, AppBarLayout child, View target, int dx, int dy, int[] consumed) 
```

```java
public void onNestedScroll(CoordinatorLayout coordinatorLayout, AppBarLayout child, View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed) 
```

```java
public void onStopNestedScroll(CoordinatorLayout coordinatorLayout, AppBarLayout child, View target) 
```

这几个方法是不是特别熟悉？我在[Android嵌套滑动机制（NestedScrolling）](http://segmentfault.com/a/1190000002873657) 介绍过，这几个方法刚好是`NestedScrollingParent`的方法，也就是对`CoodinatorLayout`进行的一个代理（Proxy），即`CoordinatorLayout`自己不对这些消息进行处理，而是传递给子view的`Behavior`，进行处理。利用这样的方法，实现了view和view之间的交互和视觉的协同（布局、滑动）。

### 总结

可以看到`CoodinatorLayout`给我们实现了一个可以被子view代理实现方法的一个布局。这和传统的`ViewGroup`不同，子view从此知道了彼此之间的存在，一个子view的变化可以通知到另一个子view。`CoordinatorLayout`所做的事情就是当成一个通信的`桥梁`，连接不同的view。使用`Behavior`对象进行通信。

我们具体的实现可以参照 Android官方文档告诉我们的每一个方法的作用 进行重写，实现自己想要的各种复杂的功能。
https://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html
有了这么一套机制，想实现组件之间的交互，就更加方便快捷啦~