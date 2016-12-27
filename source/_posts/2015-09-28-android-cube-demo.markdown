---
layout: post
title:  "Android 实现一个立方体旋转效果"
date:   2015-09-28 18:00:00 +0800
categories: development
tags: android
---

好久不见~

今天我们来看看如何实现一个立方体翻转的效果。
<!-- more -->

如图

![5dd54131gw1ewhzt3daq7g20f00qob2a.gif](http://ww2.sinaimg.cn/large/5dd54131gw1ewhzt3daq7g20f00qob2a.gif)

看上去很麻烦，实际上实现起来还是蛮轻松的。
这里我们使用到的有两个类。

1. `android.graphic.Camera` 这是在图像学概念里的摄像机，这是一个`透视摄像机`。
2. `android.graphic.Matrix` 矩阵，用来表示图像的变化。

# 头疼的钻研路开始
我们先从摄像头上的角度分析：
正常情况下，我们是这么看画面的（那个电池一样的东西就当是摄像头吧）

![clipboard.png](https://segmentfault.com/img/bVp6Nm)
我们要产生立方体的效果，那逻辑上应该是这么看：
![clipboard.png](https://segmentfault.com/img/bVp6Nv)

Camera提供了几个接口，我们这使用到的接口有两个：
1. Rotate    旋转
2. Translate 平移

这两个函数的操作都对`画布`的！
这里我们首先要有2个View。xml结构入下：
```xml
<cn.geminiwen.canvassupport.view.SplashLayout
        android:text="@string/hello_world"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:background="#000" >
        <RelativeLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:background="#f00">
        </RelativeLayout>
        <RelativeLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:background="#0f0">
        </RelativeLayout>
</cn.geminiwen.canvassupport.view.SplashLayout>
```
SplashLayout就是我的自定义布局，用来绘制立方体效果的布局。

我们把第一个view作为`backgroundView`，第二个View作为`foregroundView`，使得效果是从`backgroundView`翻转到`foregroundView`。

具体代码入下：
```java
private void cube(Canvas canvas, double interpolation) {
        View foregroundView = getChildAt(0);
        View backgroundView = getChildAt(1);
        int width = getWidth();
        int height = getHeight();
        long drawingTime = getDrawingTime();
        float rotate;

        //begin drawForeground
        rotate = (float)(- sFinalDegree * interpolation);
        mCamera.save();
        mCamera.translate((float)(width - interpolation * width), 0, 0);
        mCamera.rotateY(rotate);
        mCamera.getMatrix(mMatrix);
        mCamera.restore();

        mMatrix.postTranslate(0, height / 2);
        mMatrix.preTranslate(-width, -height / 2);
        canvas.save();
        canvas.concat(mMatrix);
        drawChild(canvas, foregroundView, drawingTime);
        canvas.restore();
        //end drawForeground



        //draw Background
        rotate = (float)(sFinalDegree - sFinalDegree * interpolation);
        mCamera.save();
        mCamera.translate((float)(-width * interpolation), 0, 0);
        mCamera.rotateY(rotate);
        mCamera.getMatrix(mMatrix);
        mCamera.restore();

        mMatrix.postTranslate(width, height / 2);
        mMatrix.preTranslate(0, -height / 2);
        canvas.save();
        canvas.concat(mMatrix);
        drawChild(canvas, backgroundView, drawingTime);
        canvas.restore();
        //end draw Background

    }
```

这段代码放到`ViewGroup`的`dispatchDraw`方法里即可，因为`ViewGroup`只能在`dispatchDraw`方法中绘制子视图。
其中，`canvas`代表画布，`interpolation`代表动画从0.0 到 1.0 的过程，方便插入器的使用。

这里来解释下我们的过程。

## View状态变换
起始状态`background`是这样的：
1. 绕Y轴正方向转90度
2. 画布x轴移动到width的位置。

可以参照上图中的`画布2`的状态。

终点状态是这样：
1. 绕Y轴正方向0度。
2. 画布x轴移动到0的位置。

可以参考上图中的画布的状态。

### 旋转问题
综上所述，我们设置转动角度`sFinalDegree`为90。
`interpolation`从0到1的过程，
`background`的rotate就变成了从`90`到`0`的过程。

### 平移问题
这时候我们考虑平移的情况，这个情况会比较复杂，因为我们这里有两种平移方式，`平移摄像机`或者`直接平移画布`。

这里我们说下区别，如果移动摄像机，会导致图像的投影发生变化，举个例子：
比如我们已经在投影上绕Y轴旋转90度，如果移动X轴的话，看如下图的区别：

1、未平移摄像机

![clipboard.png](https://segmentfault.com/img/bVp6QZ)


2、平移摄像机

![clipboard.png](https://segmentfault.com/img/bVp6Rb)
从图上我们知道，这个旋转过的画布的前端和后端我们都是可以看见的，这当然不符合我们要求，那么我们直接平移画布是什么意思呢？

我们知道对摄像机做了操作之后，应用到画布上，实际是画一个画布的投影，直接移动画布的话，就是改变其坐标系系统，达到效果，我们可以理解为同时对摄像机和view进行平移，最终达到的效果就是摄像头相对view的位置和`1`一样，但是我们的画布却平移了，这就达到了我们最终的要求。

**我们看代码虽然我们平移的是画布，但是我们平移的过程中确是使用移动摄像机的方式来绘制投影，这又是为什么呢？**

我们从三角函数的投影来解释这个问题。
首先看见我平移摄像头的方式是线性的，也就是`y=kt`这种方式，斜率一定，也就是随着时间变化，我平移的距离是线性增加的。那么考虑旋转的时候的投影情况：
被旋转的角就是`角a`，我们的画布长为`width`，那么画布的投影长度为 
`width * cos(a)`
![clipboard.png](https://segmentfault.com/img/bVp6RF)
它是一个三角函数。变化趋势先快后慢，因此我们在旋转过程中会看见右边露出背景，造成视觉上的不友好，怎么解决这个问题呢？ 

这时候就借助我们的摄像机平移的投影方式。

![clipboard.png](https://segmentfault.com/img/bVp6RT)

这里绿色的线是我们的投影线，它的投影长度比`cos(a) * width`要长很多，因此它就可以让我们在旋转过程中不产生黑边，给人视觉上的饱满感，会让我们的视觉效果好很多。

我们的`foregroundView`就是一个`backgroundView`的逆向过程，因此使用类似的代码，然后假设`interpolation`是从1-0的过程即可，同时它的坐标系整体要往左移动一个屏幕。

# 总结
做UI的效果，特别需要一些比较好的数据基础，在图像处理中，搞清楚透视、矩阵的一些计算方式和概念非常重要，今天我们介绍了利用`Camera`来进行辅助我们进行矩阵的计算。

# 源码
> demo已经放在github上
> demo已经放在github上
> demo已经放在github上
> https://github.com/geminiwen/AndroidCubeDemo 欢迎star 欢迎fork！