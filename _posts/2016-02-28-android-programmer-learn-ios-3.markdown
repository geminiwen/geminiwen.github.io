---
layout: post
title:  "Android 程序员学习 iOS —— 在线程间跳舞"
date:   2016-02-28 18:00:00 +0800
categories: android ios
---


## 跨线程更新UI

在客户端开发的过程中，我们经常碰到的问题有可能就是 IO 请求完成后，在主线程中更新 UI 这件事了，看见这个问题，我们一般会直接想到 `Handler`这个大杀器，

`Android` 中我们知道有`Looper`和`Handler`这两种神器帮我们完成不同线程间的调度，那么在`iOS`中如何实现不同线程间的切换呢？答案就是`NSOperationQueue`和`NSOperation`。变量名字直接翻译就是`操作队列`和`操作`，那么，更新 UI 就是一个`操作`。 因为`Objective-C`带了`block`和`selector`两个神器，使用闭包比在`java`中使用`Runnable`方便许多，所以我们的`NSOperation`更像是一个`Runnable`，而`NSOperationQueue`就像`Looper`和`Handler`的结合体，我们来看看如何创建一个 UI 线程上的消息队列吧。

## 主线程上的消息队列

创建一个主线程上的消息队列，只用一条函数
```
NSOperationQueue *queue = [NSOperationQueue mainQueue];
```
这和`java`创建一个`mainLooper`上的`Handler`如出一辙：

```
Handler uiHandler = new Handler(Looper.mainLooper());
```

从客户端的层面上来说，这两句话是等价的。
然后我们调用
```
[queue addOperation:...]
```
传入一个`Operation`，OS就会在适当的时候，在主线程上回调我们的`Operation`，这和`Handler`调用`handlerMessage`是如何的一致啊~

## 子线程上的消息队列
那么，如何在子线程上调用一个`NSOperation`呢？
我们知道在`Android`中，需要将`Handler`的初始化运行在子线程上，因为这样才能用`Looper.myLooper()`获取线程本地变量实例。
但是在iOS中，我们不需要显示的在一条新的线程中完成我们的工作，我们只需要使用传统的分配对象的方法：

```
NSOperationQueue *queue = [NSOperationQueue mainQueue];
```
OS这时候已经自动帮我们分配了一个消息队列（**但是不同的是，它并不像Android OS上一样，是绑定到线程上的，也就是说，这个消息队列，可以并发**），我们只需要像上面一样，使用`[queue addOperation:...]`即可。

## 实验

我们来看看实例好了，先看`mainQueue`上的实验：
```
self.queue = [NSOperationQueue mainQueue];
[self.queue addOperation:[NSBlockOperation blockOperationWithBlock:^() {
        NSLog(@"%@", [NSThread currentThread]);
}]];
```

Output:
```
2016-02-28 15:10:38.813 OperationQueue[1277:52212] <NSThread: 0x7fd44a505670>{number = 1, name = main}
```
看到我们这个`block`的执行的确是在主线程上的。
然后看新的消息队列：
```
self.queue = [[NSOperationQueue alloc] init];
[self.queue addOperation:[NSBlockOperation blockOperationWithBlock:^() {
        NSLog(@"%@", [NSThread currentThread]);
}]];
```
Output:
```
2016-02-28 15:23:03.954 OperationQueue[1375:64592] <NSThread: 0x7fe10b5136d0>{number = 2, name = (null)}
```
这就不是在主线程了，那么在`iOS`上使用"`Handler`"或者说消息队列的方法就是这么简单啦~

## One more thing —— 队列和线程的绑定

上一个章节我们说，在iOS中，线程和队列并不是一一对应绑定的，我们可以简单的理解成在`iOS`中，除了`main queue`以外，自己生成的队列是可以**并发**的，简单的操作就是在生成队列的时候，指定`maxConcurrentOperationCount`属性即可，我们的`操作队列`就具有了并发的功能（不过这样严格意义上就不是FIFO，也就是说，它其实不应该被称作"队列"了）


## 参考资料

[Working with NSOperationQueue](https://github.com/MacRuby/MacRuby/wiki/Working-with-NSOperationQueue)

[Apple官方教程 —— Concurrency Programming Guide](https://developer.apple.com/library/prerelease/ios/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091)