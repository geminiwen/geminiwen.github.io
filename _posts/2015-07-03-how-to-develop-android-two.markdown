---
layout: post
title:  "#土豆记事#教你开发Android App之 —— Hello Android"
date:   2015-07-03 18:30:10 +0800
categories: android
---

[上一篇文章](http://segmentfault.com/a/1190000002957332)，我们讲了如何创建一个工程，以及Android工程的一些基本概念，把工程创建出来后，我们看下文件目录结构，一个简单的工程结构如下。

![clipboard.png](https://segmentfault.com/img/bVmzre)
其实这个目录结构初次看还是挺让人心慌慌的。
`Android`现在引入了一个构建系统叫做[Gradle](https://gradle.org/)，你可以理解为一个C/C++里面的`Makefile` 或者是node里面的`gulp`。

`Android Studio`里面是分模块进行开发的，一个`app`可以只有一个模块，也可以有多个模块组成（比如一些自己开发的库）。如果我们的应用足够简单的话，那么就只有一个模块，`Android Studio`默认创建的模块就叫`app`，可以看见，文件夹旁边还有个小手机的标志，代表这是一个`Android Application`模块，而不是一个`Android Library`或者是其他模块。

看看我们的代码应该放哪，我们把注意力集中在这个文件夹
![clipboard.png](https://segmentfault.com/img/bVmzry)

`src`是`source`的意思，也就是源码所在的目录，我们主要就是在这个文件夹里写东西。

# Main在哪？

我们来看看，我们要的`Main`函数在哪里？
首先我们看`src/main`目录下的`AndroidManifest.xml`文件。

![clipboard.png](https://segmentfault.com/img/bVmzrM)

`AndroidManifest` 是描述App的一个最最重要的文件，一些内容的定义，主题的设置都在这里，如果熟悉`node`的朋友肯定知道`package.json`，一样一样的。

我们看到下图，在AndroidManifest中，出现的`MAIN`和`LAUNCHER`字眼，`Activity`有了他们两个的描述，它就成了你点击app的icon启动的第一个`Activity`。
![clipboard.png](https://segmentfault.com/img/bVmzr5)

在`src/main/java`文件夹中，找到`MainActivity`，打开，看见其中有一个`onCreate`的函数

![clipboard.png](https://segmentfault.com/img/bVmzsr)
顾名思义，这个函数是在这个`Activity`创建的时候调用的，它首先调用了下父类的onCreate方法（不可省略），然后调用了`setContentView`方法，这个方法是告诉Android系统：我用哪个布局文件去渲染这个`Activity`，好了，到这里一个入口的`Activity`就创建好了。

注：在`Android`系统中，`Activity`类的对象不是用来给开发者直接去`new`的，它的生命周期**由系统直接管控**因为我们不参与控制`Activity`的生命周期，因此它在什么时候回调什么函数变得异常重要。学习编程最好的去处就是官方文档，如果想更加深入了解Android Activity生命周期的童鞋，我这里推荐看下官方对它的描述 传送门：http://developer.android.com/training/basics/activity-lifecycle/index.html

# 界面如何自定义？
接下来说说`Android`中的布局系统，众所周知，`Android`一开始的设计就是为了相对布局而生的。它提供了许多强大的布局特性，我们先学习下`Android`中最常见的两种布局：
> 1. 线性布局（LinearLayout）
> 2. 相对布局（RelativeLayout）

线性布局就是子控件按顺序依次排列，线性布局可以设置方向，从上到下（vertical）或者从左到右（horizontal）。
相对布局就更自由了，如果你增加一个子控件，不设置任何属性，则子空间的位置在容器的左上角(0,0)处。如果想改变位置，可以通过在容器中的位置（比如左上，右上，左下，右下，中间，左对齐垂直居中，右对齐垂直居中等等），或者和兄弟结点的对齐方式来决定控件的位置。

布局相关的参考可以看这个链接：http://developer.android.com/training/basics/firstapp/building-ui.html

# 如何和控件交互
在`Activity`中，和`xml`相关的绑定在`setContentView`这步就算完成了。`Activity`在这之后会回调一个叫`onContentChanged`方法，在这个方法中，你可以使用类似如下代码：
```
TextView textView = findViewById(R.id.textview);
```
来获得对指定控件的引用，其中`R.id.textview`是你在`xml`中指定的`android:id`，通常情况下，在一个`xml`文件中，同样的`id`只允许出现一次。
获取到对控件的引用，我们就能调用一些控件里的方法获取我们要的内容，或者设置我们要的内容，比如我这里引用了一个`TextView`，则可以如下：
```
textView.getText()
```
获取到`textView`里面的内容。

以上就是最基础的`UI`部分的入门讲解。接下去，我们可以看看要写的App的整个结构。

> 本文提到的项目源码地址：https://github.com/geminiwen/tudounotepad
> 欢迎留言[Github](https://github.com/geminiwen)或者[@geminiwen](http://weibo.com/coffeesherk/home?wvr=5)