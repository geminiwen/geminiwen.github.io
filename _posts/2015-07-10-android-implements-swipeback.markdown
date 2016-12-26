---
layout: post
title:  "Android实现SwipeBack（右滑退出）效果"
date:   2015-07-10 18:00:00 +0800
categories: android
---


## 效果演示
### 初始状态

![clipboard.png](https://segmentfault.com/img/bVmEJE)

### 滑动中状态
![clipboard.png](https://segmentfault.com/img/bVmEJF)
### 结束状态

![clipboard.png](https://segmentfault.com/img/bVmEJK)

这是目前实现在SegmentFault for Android v2.6中的效果。
一切一切的之前，感谢 [ikew0ng/SwipeBackLayout](https://github.com/ikew0ng/SwipeBackLayout)
我使用这个库，并经过一些修改，支持了Android 4.0以上所有的版本。
我们来分析下`SwipeBackLayout`的源码

## 一些修改

我之前做过实验，碰到的最大问题是上层的Activity底下并不是透明的，因此看不见下层Activity的视图。
在`SwipeBackLayout`中采用的方案是使用一个叫`convertToTranslucent`的未公开的api，再配合`theme`中
把`windowIsTranslucent`设置为`true`，即可实现上层的`Window`背景为透明。

这里要注意的地方是调用`convertToTranslucent`可以使用反射的方法进行调用，但是在`Lollipop`中，它的参数变成了两个，而在5.0以下是一个参数，所以需要在源码中对`Util.convertActivityToTranslucent`这个方法进行一些修改。

```java
public static void convertActivityToTranslucent(Activity activity) {
    try {
        Class[] t = Activity.class.getDeclaredClasses();
        Class translucentConversionListenerClazz = null;
        Class[] method = t;
        int len$ = t.length;

        for(int i$ = 0; i$ < len$; ++i$) {
            Class clazz = method[i$];
            if(clazz.getSimpleName().contains("TranslucentConversionListener")) {
                translucentConversionListenerClazz = clazz;
                break;
            }
        }
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            Method var8 = Activity.class.getDeclaredMethod("convertToTranslucent", translucentConversionListenerClazz, ActivityOptions.class);
            var8.setAccessible(true);
            var8.invoke(activity, new Object[]{null, null});
        } else {
            Method var8 = Activity.class.getDeclaredMethod("convertToTranslucent", translucentConversionListenerClazz);
            var8.setAccessible(true);
            var8.invoke(activity, new Object[]{null});
        }
    } catch (Throwable e) {
    }

}
```

使得能适配4.0 - 5.0+所有的设备

## 源码分析
我们可以看它的源码可以知道，它是利用`ViewDragHelper`对View进行拖移效果的，`ViewDragHelper`主要帮助我们完成了对速度、加速度以及释放后的一些逻辑的设置，大大简化了我们对触摸事件的处理。
我们看下`SwipeBackLayout`是如何嵌入到我们要的`Activity`里去的
```
 public void attachToActivity(Activity activity) {
    this.mActivity = activity;
    // .... 省略部分代码
    ViewGroup decor = (ViewGroup)activity.getWindow().getDecorView();
    ViewGroup decorChild = (ViewGroup)decor.getChildAt(0);
    decorChild.setBackgroundResource(background);
    decor.removeView(decorChild);
    this.addView(decorChild);
    this.setContentView(decorChild);
    decor.addView(this);
}
```

我们可以看到，本来`Activity`调用`setContentView`之后，会把我们要的`layout`加到`window`的`decorView`上，我们在这里把`window`中的`decorView`的子元素改成`SwipeBackLayout`，然后把原先的`contentView`加到`decorView`下，使得`ViewDragHelper`附着在`SwipeBackLayout`上。
```
this.mDragHelper = ViewDragHelper.create(this, new SwipeBackLayout.ViewDragCallback());
```
`ViewDragHelper`使用`ViewDragHelper.Callback`这个回调进行一些View是否可以滑动以及滑动距离的判定。
我们来看下`ViewDragHelper.Callback`接口的声明
官方文档传送门：http://developer.android.com/reference/android/support/v4/widget/ViewDragHelper.Callback.html
|定义|注释|
|--|--|
|int clampViewPositionHorizontal (View child, int left, int dx)|此方法返回一个值，告诉Helper，这个view能滑动的最大（或者负向最大）的横向坐标|
|int clampViewPositionVertical (View child, int top, int dy)|此方法返回一个值，告诉Helper，这个view能滑动的最大（或者负向最大）的纵向坐标|
|int getOrderedChildIndex (int index)|返回这个索引所指向的子视图的Z轴坐标|
|int getViewHorizontalDragRange (View child)|返回指定View在横向上能滑动的最大距离|
|int getViewVerticalDragRange (View child)|返回指定View在纵向上能滑动的最大距离|
|void onEdgeDragStarted (int edgeFlags, int pointerId)|当边缘开始拖动的时候，会调用这个回调|
|boolean onEdgeLock (int edgeFlags)|返回指定的边是否被锁定|
|void onEdgeTouched (int edgeFlags, int pointerId)|当边缘被触摸时，系统会回调这个函数|
|void onViewCaptured (View capturedChild, int activePointerId)|当有一个子视图被指定为可拖动时，系统会回调这个函数|
|void onViewDragStateChanged (int state)|拖动状态改变时，会回调这个函数|
|void onViewPositionChanged (View changedView, int left, int top, int dx, int dy)|当子视图位置变化时，会回调这个函数|
| void onViewReleased (View releasedChild, float xvel, float yvel)|当手指从子视图松开时，会调用这个函数，同时返回在x轴和y轴上当前的速度|
| boolean tryCaptureView (View child, int pointerId)|系统会依次列出这个父容器的子视图，你需要指定当前传入的这个视图是否可拖动，如果可拖动则返回`true` 否则为`false`|

利用`ViewDragHelper`的回调函数，我们知道了视图被拖动的距离，然后根据这个距离和宽度的一些比例，我们可以对`SwipeBackLayout`的父容器进行一些透明度和阴影的设置。

实现这个效果就是这么简单~

> 欢迎关注我[Github](https://github.com/geminiwen) 以及 [@Gemini](http://weibo.com/coffeesherk/home?leftnav=1) 