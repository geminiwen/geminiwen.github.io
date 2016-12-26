---
layout: post
title:  "谜之RxJava （三）update —— 线程切换（二）"
date:   2016-04-03 18:00:00 +0800
categories: android
---

在`RxJava`更新版本后，`OperatorSubscribeOn`这个接口进行了一个重构，变换方式从一个比较难理解的递归嵌套的`Observable<Observable<T>>`上做一个`Operator`改成了从`OnSubscribe`角度上进行了一层封装。

从类型来说，`OperatorSubscribeOn`脱离了`Operator`的概念，变身成了`OnSubscribe`。

我们来比对下吧~

老版本的核心实现:
```
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

> 操作符，核心是把一个`Subscriber`转换成另外一个`Subscriber`

再看看新版实现
```
@Override
public void call(final Subscriber<? super T> subscriber) {
    final Worker inner = scheduler.createWorker();
    subscriber.add(inner);
    
    inner.schedule(new Action0() {
        @Override
        public void call() {
            final Thread t = Thread.currentThread();
            
            Subscriber<T> s = new Subscriber<T>(subscriber) {
                @Override
                public void onNext(T t) {
                    subscriber.onNext(t);
                }
                
                @Override
                public void onError(Throwable e) {
                    try {
                        subscriber.onError(e);
                    } finally {
                        inner.unsubscribe();
                    }
                }
                
                @Override
                public void onCompleted() {
                    try {
                        subscriber.onCompleted();
                    } finally {
                        inner.unsubscribe();
                    }
                }
                
                @Override
                public void setProducer(final Producer p) {
                    subscriber.setProducer(new Producer() {
                        @Override
                        public void request(final long n) {
                            if (t == Thread.currentThread()) {
                                p.request(n);
                            } else {
                                inner.schedule(new Action0() {
                                    @Override
                                    public void call() {
                                        p.request(n);
                                    }
                                });
                            }
                        }
                    });
                }
            };
            
            source.unsafeSubscribe(s);
        }
    });
}
```

这里实现的是`OnSubscribe`接口，我们知道，`OnSubscribe`是`Observable`真正执行的代码段。

在新的接口重构后，唯一的不同是，在它里面需要存一个指向原始`Observable`的source变量。 而在老接口中，变换前的`Observable`是通过`Observable<Observable>`传进来的。

欢迎关注[我的专栏](https://segmentfault.com/blog/gemini)，来从头到尾学习RxJava