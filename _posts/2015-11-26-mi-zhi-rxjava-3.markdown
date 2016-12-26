---
layout: post
title:  "谜之RxJava （三）—— 线程切换"
date:   2015-11-26 18:00:00 +0800
categories: android
---

[【谜之RxJava （二） —— Magic Lift】](http://segmentfault.com/a/1190000004049841)

## Rxjava -- 一个异步库
`RxJava`最迷人的是什么？
答案就是`把异步序列写到一个工作流里！`和`javascript`的`Promise/A`如出一辙。
OK，在`java`中做异步的事情在我们传统理解过来可不方便，而且，如果要让异步按照我们的工作流来，就更困难了。

但是在`RxJava`中，我们只要调用调用
`subscribOn()`和`observeOn()`就能切换我们的工作线程，是不是让小伙伴都惊呆了？

然后结合`RxJava`的`Operator`，写异步的时候，想切换线程就是一行代码的事情，整个`workflow`还非常清晰：

```java
Observable.create()
// do something on io thread
.work() // work.. work..
.subscribeOn(Schedulers.io())
// observeOn android main thread
.observeOn(AndroidSchedulers.mainThread())
.subscribe();
```

我们再也不用去写什么见鬼的`new Thread`和`Handler`了，在这么几行代码里，我们实现了在`io`线程上做我们的工作(`work`)，在`main`线程上，更新UI

## Subscribe On
先看下`subscribeOn`干了什么
```
 public final Observable<T> subscribeOn(Scheduler scheduler) {
    if (this instanceof ScalarSynchronousObservable) {
        return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
    }
    return nest().lift(new OperatorSubscribeOn<T>(scheduler));
}
```
啊，原来也是个lift，就是从一个`Observable`生成另外一个`Observable`咯，这个`nest`是干嘛用？
```java
 public final Observable<Observable<T>> nest() {
    return just(this);
}
```
这里返回类型告诉我们，它是产生一个`Observable<Observable<T>>`
讲到这里，会有点晕，先记着这个，然后我们看`OperatorSubscribeOn`这个操作符,

构造函数是

```java
public OperatorSubscribeOn(Scheduler scheduler) {
    this.scheduler = scheduler;
}
```

OK，这里保存了`scheduler`对象，然后就是我们前一章说过的转换方法。
```java
 @Override
public Subscriber<? super Observable<T>> call(final Subscriber<? super T> subscriber) {
    final Worker inner = scheduler.createWorker();
    subscriber.add(inner);
    return new Subscriber<Observable<T>>(subscriber) {

        @Override
        public void onCompleted() {
            // ignore because this is a nested Observable and we expect only 1 Observable<T> emitted to onNext
        }

        @Override
        public void onError(Throwable e) {
            subscriber.onError(e);
        }

        @Override
        public void onNext(final Observable<T> o) {
            inner.schedule(new Action0() {

                @Override
                public void call() {
                    final Thread t = Thread.currentThread();
                    o.unsafeSubscribe(new Subscriber<T>(subscriber) {

                        @Override
                        public void onCompleted() {
                            subscriber.onCompleted();
                        }

                        @Override
                        public void onError(Throwable e) {
                            subscriber.onError(e);
                        }

                        @Override
                        public void onNext(T t) {
                            subscriber.onNext(t);
                        }

                        @Override
                        public void setProducer(final Producer producer) {
                            subscriber.setProducer(new Producer() {

                                @Override
                                public void request(final long n) {
                                    if (Thread.currentThread() == t) {
                                        // don't schedule if we're already on the thread (primarily for first setProducer call)
                                        // see unit test 'testSetProducerSynchronousRequest' for more context on this
                                        producer.request(n);
                                    } else {
                                        inner.schedule(new Action0() {

                                            @Override
                                            public void call() {
                                                producer.request(n);
                                            }
                                        });
                                    }
                                }

                            });
                        }

                    });
                }
            });
        }

    };
}
```

## 让人纠结的类模板

看完这段又臭又长的，先深呼吸一口气，我们慢慢分析下。
首先要注意`RxJava`里面最让人头疼的模板问题，那么`OperatorMap`这个类的声明是
```java
public final class OperatorMap<T, R> implements Operator<R, T>
```
而`Operator`这个接口继承`Func1`
```java
public interface Func1<T, R> extends Function {
    R call(T t);
}
```
我们这里不要记`T`和`R`，记住`传入左边的模板是形参，传入右边的模板是返回值`。

> 好了，那么这里的`call`就是从一个`T`转换成一个`Observable<T>`的过程了。

总结一下，我们这一次调用`subscribeOn`，做了两件事

> 1、`nest()` 为`Observable<T>`生成了一个`Observable<Observable<T>>`        
> 2、`lift()` 对`Observalbe<Observalbe<T>>`进行一个变化，变回`Observable<T>`

因为`lift`是一个模板函数，它的返回值的类型是参照它的形参来，而他的形参是`Operator<T, Observable<T>>` 这个结论非常重要！！
OK，到这里我们已经存储了所有的序列，等着我们调用了。

## 调用链

首先，记录我们在调用这条指令之前的`Observable<T>`，记为`Observable$1`
然后，经过`lift`生成的`Observable<T>`记为`Observable$2`

好了，现在我们拿到的依然是`Observable<T>`这个对象，但是它**不是**原始的`Observable$1`，要深深记住这一点，它是由`lift`生成的`Observable$2`，这时候进行`subscribe`，那看到首先调用的就是`OnSubscribe.call`方法，好，直接进入`lift`当中生成的那个地方。

我们知道这一层`lift`的`operator`就是刚刚的`OperatorSubscribOn`，那么调用它的`call`方法，生成的是一个`Subscriber<Observable<T>>`
```java
Subscriber<? super T> st = hook.onLift(operator).call(o);
try {
    // new Subscriber created and being subscribed with so 'onStart' it
    st.onStart();
    onSubscribe.call(st);
} catch (Throwable e) {
...
}
```
好，还记得我们调用过`nest`么？，这里的`onSubscribe`可是`nest`上下文中的噢，每一次，到这个地方，这个`onSubscribe`就是上一层`Observable`的`onSubscribe`，即`Observable<Observable<T>>`的`onSubscribe`，相当于栈弹出了一层。它的`call`直接在`Subscriber`的`onNext`中给出了最开始的`Observable<T>`，我们这里就要看下刚刚在`OperatorSubscribeOn`中生成的`Subscriber`

```java
new Subscriber<Observable<T>>(subscriber) {

    @Override
    public void onCompleted() {
        // ignore because this is a nested Observable and we expect only 1 Observable<T> emitted to onNext
    }
    
    @Override
    public void onError(Throwable e) {
        subscriber.onError(e);
    }
    
    @Override
    public void onNext(final Observable<T> o) {
        inner.schedule(new Action0() {
    
            @Override
            public void call() {
                final Thread t = Thread.currentThread();
                o.unsafeSubscribe(new Subscriber<T>(subscriber) {
    
                    @Override
                    public void onCompleted() {
                        subscriber.onCompleted();
                    }
    
                    @Override
                    public void onError(Throwable e) {
                        subscriber.onError(e);
                    }
    
                    @Override
                    public void onNext(T t) {
                        subscriber.onNext(t);
                    }
                });
            }
        });
    }
}
```

对，就是它，这里要注意，这里的`subscriber`就是我们在`lift`中，传入的`o`
```java
Subscriber<? super T> st = hook.onLift(operator).call(o);
```
对，就是它，其实它就是`SafeSubscriber`。

回过头，看看刚刚的`onNext()`方法，`inner.schedule()` 这个函数，我们可以认为就是`postRun()`类似的方法，而`onNext()`中传入的`o`是我们之前生成的`Observable$1`，是从`Observable.just`封装出来的`Observable<Observable<T>>`中产生的，这里调用了`Observable$1.unsafeSubscribe`方法，我们暂时不关心它和`subscribe`有什么不同，但是我们知道最终功能是一样的就好了。

> 注意它运行时的线程！！在`inner`这个`Worker`上！于是它的运行线程已经被改了！！

好，这里的`unsafeSubscribe`调用的方法就是调用原先`Observable$1.onSubscribe`中的`call`方法：
这个`Observable$1`就是我们之前自己定义的`Observable`了。

综上所述，如果我们需要我们的`Observable$1`在一个别的线程上运行的时候，只需要在后面跟一个`subscribeOn`即可。结合扔物线大大的图如下：
![rxjavarxjava_12.png][1]

## 总结

这里逻辑着实不好理解。如果还没有理解的朋友，可以按照我前文说的顺序，细致的看下来，我把逻辑过一遍之后，发现`lift`的陷阱实在太大，内部类用的风生水起，一不小心，就不知道一个变量的上下文是什么，需要特别小心。


[迷之RxJava（四）—— Retrofit和RxJava的基情](http://segmentfault.com/a/1190000004077117)

> 本文在不停更新中，如果有不明白的地方（可能会有很多），请大家给出意见，拍砖请轻点= =


  [1]: /img/bVq93J