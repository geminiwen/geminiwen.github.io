---
layout: post
title:  "谜之RxJava （二） ——  Magic Lift"
date:   2015-11-26 18:00:00 +0800
categories: android
---

## 回顾
[上一篇文章](http://segmentfault.com/a/1190000004049490) 讲了`Observable`、`OnSubscribe`和`Subscriber`之间的关系。 我们知道，`Observable`的具体工作都是在`OnSubscribe`中完成的。从这个类名我们也知道，如果生成了一个`Observable`对象，而不进行`subscribe`，那么什么都不会发生！

OK，`RxJava`最让人兴奋的就是它有各种各样的操作符，什么`map`呀，`flatMap`呀各种，我们今天要`知其然知其所以然`，那么他们是如何实现功能的呢？

## 例子
```java
Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("hello");
    }
})
.map(new Func1<String, String>() {
    @Override
    public String call(String s) {
        return s + "word";
    }
})
.subscribe(new Subscriber<String>() {
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

## lift
我们先看下进行链式调用`map`之后，发生了什么。
```java
public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
    return lift(new OperatorMap<T, R>(func));
}
```
对，就是调用了`lift`函数！，然后把我们的转换器（Transfomer，我好想翻译成变形金刚）传入进去，看下它做了什么事。
```java
public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
    return new Observable<R>(new OnSubscribe<R>() {
        @Override
        public void call(Subscriber<? super R> o) {
            try {
                Subscriber<? super T> st = hook.onLift(operator).call(o);
                try {
                    // new Subscriber created and being subscribed with so 'onStart' it
                    st.onStart();
                    onSubscribe.call(st);
                } catch (Throwable e) {
                    // localized capture of errors rather than it skipping all operators 
                    // and ending up in the try/catch of the subscribe method which then
                    // prevents onErrorResumeNext and other similar approaches to error handling
                    if (e instanceof OnErrorNotImplementedException) {
                        throw (OnErrorNotImplementedException) e;
                    }
                    st.onError(e);
                }
            } catch (Throwable e) {
                if (e instanceof OnErrorNotImplementedException) {
                    throw (OnErrorNotImplementedException) e;
                }
                // if the lift function failed all we can do is pass the error to the final Subscriber
                // as we don't have the operator available to us
                o.onError(e);
            }
        }
    });
}
```

来，我来简化一下
```java
public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
    return new Observable<R>(...);
}
```
返回了一个新的`Observable`对象，这才是重点！ 这种链式调用看起来特别熟悉？有没有像`javascript`中的`Promise/A`，在`then`中返回一个`Promise`对象进行链式调用？

OK，那么我们要看下它是如何工作的啦。

在`map()`调用之后，我们操作的就是新的`Observable`对象，我们可以把它取名为`Observable$2`，OK，我们这里调用`subscribe`，完整的就是`Observable$2.subscribe`，继续看到`subscribe`里，重要的几个调用：
```java
hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
return hook.onSubscribeReturn(subscriber);
```

> 注意注意 ！ 这里的`observable`是`Observable$2`！！也就是说，这里的`onSubscribe`是，`lift`中定义的！！

OK，我们追踪下去，回到`lift`的定义中。
```java
return new Observable<R>(new OnSubscribe<R>() {
    @Override
    public void call(Subscriber<? super R> o) {
        try {
            Subscriber<? super T> st = hook.onLift(operator).call(o);
            try {
                // new Subscriber created and being subscribed with so 'onStart' it
                st.onStart();
                onSubscribe.call(st); //请注意我！！ 这个onSubscribe是原始的OnSubScribe对象！！
            } catch (Throwable e) {
                // localized capture of errors rather than it skipping all operators 
                // and ending up in the try/catch of the subscribe method which then
                // prevents onErrorResumeNext and other similar approaches to error handling
                if (e instanceof OnErrorNotImplementedException) {
                    throw (OnErrorNotImplementedException) e;
                }
                st.onError(e);
            }
        } catch (Throwable e) {
            if (e instanceof OnErrorNotImplementedException) {
                throw (OnErrorNotImplementedException) e;
            }
            // if the lift function failed all we can do is pass the error to the final Subscriber
            // as we don't have the operator available to us
            o.onError(e);
        }
    }
});
```
一定一定要注意这段函数执行的上下文！，这段函数中的`onSubscribe`对象指向的是外部类，也就是第一个`Observable`的`onSubScribe`！而不是`Observable$2`中的`onSubscribe`，OK，谨记这一点之后，看看
```java
Subscriber<? super T> st = hook.onLift(operator).call(o);
```
这行代码，就是定义`operator`，生成一个经过`operator`操作过的`Subscriber`，看下`OperatorMap`这个类中的`call`方法
```java
@Override
public Subscriber<? super T> call(final Subscriber<? super R> o) {
    return new Subscriber<T>(o) {

        @Override
        public void onCompleted() {
            o.onCompleted();
        }

        @Override
        public void onError(Throwable e) {
            o.onError(e);
        }

        @Override
        public void onNext(T t) {
            try {
                o.onNext(transformer.call(t));
            } catch (Throwable e) {
                Exceptions.throwIfFatal(e);
                onError(OnErrorThrowable.addValueAsLastCause(e, t));
            }
        }

    };
}
```
没错，对传入的`Subscriber`做了一个代理，把转换后的值传入。
这样就生成了一个代理的`Subscriber`，

最后我们最外层的`OnSubscribe`对象对我们代理的`Subscriber`进行了调用。。
也就是
```java
 @Override
public void call(Subscriber<? super String> subscriber) {
    //此处的subscriber就是被map包裹(wrapper)后的对象。
    subscriber.onNext("hello");
}
```
然后这个`subscriber`传入到内部，链式的通知，最后通知到我们在`subscribe`函数中定义的对象。

这时候要盗下扔物线大大文章的图
![rxjavarxjava_8_d.gif][1]

还不明白的各位，可以自己写一个Demo试一下。

下一章讲下`RxJava`中很重要的线程切换。

[【迷之RxJava（三）—— 线程切换】](http://segmentfault.com/a/1190000004051191)

> 欢迎关注我[Github](https://github.com/geminiwen) 以及 [weibo](http://weibo.com/coffeesherk/home?leftnav=1)、[@Gemini](/u/gemini)
  [1]: /img/bVq9H0