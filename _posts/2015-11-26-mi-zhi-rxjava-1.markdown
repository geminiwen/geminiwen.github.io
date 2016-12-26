---
layout: post
title:  "谜之RxJava （一） —— 最基本的观察者模式"
date:   2015-11-30 18:00:00 +0800
categories: android
---

最近在`Android`界，最火的framework大概就是`RxJava`了。
扔物线大大之前写了一篇文章 [《给 Android 开发者的 RxJava 详解》](http://gank.io/post/560e15be2dca930e00da1083)，在我学习RxJava的过程中受益匪浅。经过阅读这篇文章后，我们来看下`RxJava`的源码，揭开它神秘的面纱。

这里准备分几篇文章写，为了能让自己有个喘口气的机会。

先来上个最最简单的，经典的Demo。

## Demo
```java
Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("hello");
    }
}).subscribe(new Subscriber<String>() {
    @Override
    public void onCompleted() {

    }

    @Override
    public void onError(Throwable e) {

    }

    @Override
    public void onNext(String s) {
        Log.d("rx", s);
    }
});
```
这段代码产生的最终结果就是在Log里会出现hello。

看下这段代码的具体流程吧。
这里有2个函数`create`和`subscribe`，我们看看`create`里面干了啥。

## OnSubscribe对象
```java
public final static <T> Observable<T> create(OnSubscribe<T> f) {
    return new Observable<T>(hook.onCreate(f));
}
// constructor
protected Observable(OnSubscribe<T> f) {
    this.onSubscribe = f;
}
```

这里的`hook`是一个默认实现，里面不做任何事，就是返回`f`。我们看见`create`只是给`Observable`的`onSubscribe`赋值了我们定义的`OnSubscribe`。

## Subscriber对象

来看下`subscribe`这个函数做了什么事
```java
public final Subscription subscribe(Subscriber<? super T> subscriber) {
    return Observable.subscribe(subscriber, this);
}

private static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
 // validate and proceed
    if (subscriber == null) {
        throw new IllegalArgumentException("observer can not be null");
    }
    if (observable.onSubscribe == null) {
        throw new IllegalStateException("onSubscribe function can not be null.");
        /*
         * the subscribe function can also be overridden but generally that's not the appropriate approach
         * so I won't mention that in the exception
         */
    }
    
    // new Subscriber so onStart it
    subscriber.onStart();
    
    /*
     * See https://github.com/ReactiveX/RxJava/issues/216 for discussion on "Guideline 6.4: Protect calls
     * to user code from within an Observer"
     */
    // if not already wrapped
    if (!(subscriber instanceof SafeSubscriber)) {
        // assign to `observer` so we return the protected version
        subscriber = new SafeSubscriber<T>(subscriber);
    }

    // The code below is exactly the same an unsafeSubscribe but not used because it would add a sigificent depth to alreay huge call stacks.
    try {
        // allow the hook to intercept and/or decorate
        hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
        return hook.onSubscribeReturn(subscriber);
    } catch (Throwable e) {
        // special handling for certain Throwable/Error/Exception types
        Exceptions.throwIfFatal(e);
        // if an unhandled error occurs executing the onSubscribe we will propagate it
        try {
            subscriber.onError(hook.onSubscribeError(e));
        } catch (OnErrorNotImplementedException e2) {
            // special handling when onError is not implemented ... we just rethrow
            throw e2;
        } catch (Throwable e2) {
            // if this happens it means the onError itself failed (perhaps an invalid function implementation)
            // so we are unable to propagate the error correctly and will just throw
            RuntimeException r = new RuntimeException("Error occurred attempting to subscribe [" + e.getMessage() + "] and then again while trying to pass to onError.", e2);
            // TODO could the hook be the cause of the error in the on error handling.
            hook.onSubscribeError(r);
            // TODO why aren't we throwing the hook's return value.
            throw r;
        }
        return Subscriptions.unsubscribed();
    }
}
```
我们看到，这里我们的`subscriber`被`SafeSubscriber`包裹了一层。
```java
if (!(subscriber instanceof SafeSubscriber)) {
    // assign to `observer` so we return the protected version
    subscriber = new SafeSubscriber<T>(subscriber);
}
```
然后开始执行工作流
```java
hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
return hook.onSubscribeReturn(subscriber);
```
默认的`hook`只是返回我们之前定义的`onSubscribe`，这里调用的`call`方法就是我们在外面定义的
```java
new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("hello");
    }
})
```
我们调用传入的`subscriber`对象的`onNext`方法，这里的`subscriber`是`SafeSubscriber`
在`SafeScriber`中
```java
public void onNext(T args) {
    try {
        if (!done) {
            actual.onNext(args);
        }
    } catch (Throwable e) {
        // we handle here instead of another method so we don't add stacks to the frame
        // which can prevent it from being able to handle StackOverflow
        Exceptions.throwIfFatal(e);
        // handle errors if the onNext implementation fails, not just if the Observable fails
        onError(e);
    }
}
```
`actual`就是我们自己定义的`subscriber`。 原来`SafeSubscriber`只是为了帮我们处理好异常，以及防止工作流的重复。

这是`RxJava`最最基本的工作流，让我们认识到他是怎么工作的。之后我们来讲讲其中的细节和其他神奇的内容。

[【谜之RxJava （二） —— Magic Lift】](http://segmentfault.com/a/1190000004049841)


> 欢迎关注我[Github](https://github.com/geminiwen) 以及 [weibo](http://weibo.com/coffeesherk/home?leftnav=1)、[@Gemini](/u/gemini)
