---
layout: post
title:  "#土豆记事# ——学习Kotlin（Android中的Swift）"
date:   2015-07-13 18:00:00 +0800
categories: android
---

# 概览

之前我们学习过如何写一个简单的Android App。
为了赶上潮流，我特地去学习了下**Jetbrains**开发的新语言 —— Kotlin

不想说太多的概念，总结出来就是 `Swift on JVM`。
那么为什么要用它呢，我喜欢它的理由很多：
> 1. 带来了**Nullable Safe**特性 —— 以后再也不怕讨厌的 Null Pointer Exception了。
> 2. 闭包闭包闭包 —— 重要的事情说三遍.
> 3. Smart Type Case —— 很智能的一个特性，当你使用if检查是否是某种类型以后，自动转换为指定类型。
> 4. 没有附加的Runtime —— iOSer 看到这会不会哭.
> 5. Kotlin stdlib 非常小，打包后Apk的体积几乎没有变化，也不用担心方法数超过限制。

总而言之，就是用极小的代价换来了我们许多振奋人心的特性，那么你心动了么？
当然，在心动之前也要理智，我们要知道Kotlin暂时还没有发布"正式版"，一直在`0.x.x`版本号中徘徊，如果你足够胆大（像我），那么你大可以一试。

>老规矩先补上官方文档传送门：http://kotlinlang.org/docs/reference/basic-syntax.html



一些基本的语法如——基础类型、流程控制、类与继承等等特性我们已经不陌生，我们来看看几个新特性。

# Nullable Safe
我翻译过来是空指针安全监测，什么意思呢？看下如下语法糖（Swift Developer可以直接跳过）
比如
```
var text:String? = null
text?.length()
```
如果是`java`代码，在`text`变量为`null`的时候，调用`text.length()`是会崩溃滴，那么在这里，我们用了`?`来告诉编译器，如果`text`为`null`，则返回`null`，否则返回`text.length()`，具体翻译过来就是这样：
```
if (text == null) {
   return null;
} else {
   return text.length()
}
```
看我们少了这么多判断，这个语法糖是不是很棒？

# Closure & Lambda & Higer-Order Functions
介绍我最爱的闭包，那么在java 8以下的版本中，java是没有闭包这个特性的，举个例子，你的函数不能被当成一个对象使用，必须使用一个接口封装，而我们使用`Kotlin`就可以直接传函数当形参啦！

```
fun lock<T>(lock: Lock, body: () -> T): T {
  lock.lock()
  try {
    return body()
  }
  finally {
    lock.unlock()
  }
}
```
此处我们的`body`形参就是一个0参数返回类型为T的函数，可以作为我们的回调函数使用，而不用像java一样定义又臭又长的接口，再传入使用。

```
fun dfs(graph: Graph) {
  fun dfs(current: Vertex, visited: Set<Vertex>) {
    if (!visited.add(current)) return
    for (v in current.neighbors)
      dfs(v, visited)
  }

  dfs(graph.vertices[0], HashSet())
}
```
这是一个典型的dfs算法，使用Kotlin的高阶函数特性——可以在函数内定义函数。

# Smart Type Case
智能类型转换，什么意思呢？show一段代码
```
if (str is String) {
    return str.length()
} else if (str is Int) {
    return str.toString()
}
```
我们知道Int类型是没有`length`这个方法的，也就是经过这个if判断，如果满足if条件的话，编译器自动帮我们转换成我们要的类型，然后供我们调用。

我试着在**SegmentFault for Android**中加入对`Kotlin`的支持，在加入`Kotlin`的`lib`前后，包大小并没有明显增长（1M以下），性能亦没有降低，所以用户是感知不出来内部发生了什么变化。
综上所述，我要给`Kotlin`点赞！

# Demo
我把**土豆记事**所有的代码全部改成`Kotlin`的实现，并开源到`Github`上 
https://github.com/geminiwen/tudounotepad
大家可以clone下来学习，也非常感谢大家对我的支持。
**（顺便跪求各种star star star)**

> 欢迎关注我[Github](https://github.com/geminiwen) 以及 [@Gemini](http://weibo.com/coffeesherk/home?leftnav=1) 