---
layout: post
title:  "谜之RxJava （三）update 2 —— subscribeOn 和 observeOn 的区别"
date:   2016-04-03 18:00:00 +0800
categories: development
tags: android
---

## 开头

之前我们分析过`subscribeOn`这个函数，
现在我们来看下`subscribeOn`和`observeOn`这两个函数到底有什么异同。

用过`rxjava`的旁友都知道，`subscribeOn`和`observeOn`都是用来切换线程用的，可是我什么时候用`subscribeOn`，什么时候用`observeOn`呢，我们很少知道这两个区别是啥。

> 友情提示，如果不想看分析过程的，可以直接跳到下面的总结部分。

## subscribeOn

先看下`OperatorSubscribeOn`的核心代码：
```
public final class OperatorSubscribeOn<T> implements OnSubscribe<T> {

    final Scheduler scheduler;
    final Observable<T> source;

    public OperatorSubscribeOn(Observable<T> source, Scheduler scheduler) {
        this.scheduler = scheduler;
        this.source = source;
    }

    @Override
    public void call(final Subscriber<? super T> subscriber) {
        final Worker inner = scheduler.createWorker();
        subscriber.add(inner);
        
        inner.schedule(new Action0() {
        
            @Override
            public void call() {
               
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
                    
                    ....
                };
                
                source.unsafeSubscribe(s);
            }
        });
    }
}
```

这里注意两点：
> 1. 因为`OperatorSubscribeOn`是个`OnSubscribe`对象，所以在`call`参数中传入的`subscriber`就是我们在外面使用`Observable.subscribe(a)`传入的对象`a`。

> 2. 这里`source`对象指向的是调用`subscribeOn`之前的那个`Observable`序列。

明确了这两点，我们就很好的知道了`subscribeOn`是如何工作，产生神奇的效果了。
其实最最主要的就是一行函数
> source.unsafeSubscribe(s);

并且要注意它所在的位置，是在worker的`call`里面，说白了，就是把`source.subscribe`这一行调用放在指定的线程里，那么总结起来的结论就是：

> `subscribeOn`的调用，改变了调用前序列所运行的线程。

## observeOn

同样看下`OperatorObserveOn`这个类的主要代码:
```
public final class OperatorObserveOn<T> implements Operator<T, T> {

    private final Scheduler scheduler;
    private final boolean delayError;

    /**
     * @param scheduler the scheduler to use
     * @param delayError delay errors until all normal events are emitted in the other thread?
     */
    public OperatorObserveOn(Scheduler scheduler, boolean delayError) {
        this.scheduler = scheduler;
        this.delayError = delayError;
    }

    @Override
    public Subscriber<? super T> call(Subscriber<? super T> child) {
        ....
        ObserveOnSubscriber<T> parent = new ObserveOnSubscriber<T>(scheduler, child, delayError);
        parent.init();
        return parent;
    }

    /** Observe through individual queue per observer. */
    private static final class ObserveOnSubscriber<T> extends Subscriber<T> implements Action0 {
        final Subscriber<? super T> child;
        final Scheduler.Worker recursiveScheduler;
        final NotificationLite<T> on;
        final boolean delayError;
        final Queue<Object> queue;
        
        // the status of the current stream
        volatile boolean finished;

        final AtomicLong requested = new AtomicLong();
        
        final AtomicLong counter = new AtomicLong();
        
        /** 
         * The single exception if not null, should be written before setting finished (release) and read after
         * reading finished (acquire).
         */
        Throwable error;

        // do NOT pass the Subscriber through to couple the subscription chain ... unsubscribing on the parent should
        // not prevent anything downstream from consuming, which will happen if the Subscription is chained
        public ObserveOnSubscriber(Scheduler scheduler, Subscriber<? super T> child, boolean delayError) {
            this.child = child;
            this.recursiveScheduler = scheduler.createWorker();
            this.delayError = delayError;
            this.on = NotificationLite.instance();
            if (UnsafeAccess.isUnsafeAvailable()) {
                queue = new SpscArrayQueue<Object>(RxRingBuffer.SIZE);
            } else {
                queue = new SpscAtomicArrayQueue<Object>(RxRingBuffer.SIZE);
            }
        }
        
        void init() {
            // don't want this code in the constructor because `this` can escape through the 
            // setProducer call
            Subscriber<? super T> localChild = child;
            
            localChild.setProducer(new Producer() {

                @Override
                public void request(long n) {
                    if (n > 0L) {
                        BackpressureUtils.getAndAddRequest(requested, n);
                        schedule();
                    }
                }

            });
            localChild.add(recursiveScheduler);
            localChild.add(this);
        }

        @Override
        public void onStart() {
            // signal that this is an async operator capable of receiving this many
            request(RxRingBuffer.SIZE);
        }

        @Override
        public void onNext(final T t) {
            if (isUnsubscribed() || finished) {
                return;
            }
            if (!queue.offer(on.next(t))) {
                onError(new MissingBackpressureException());
                return;
            }
            schedule();
        }

        @Override
        public void onCompleted() {
            if (isUnsubscribed() || finished) {
                return;
            }
            finished = true;
            schedule();
        }

        @Override
        public void onError(final Throwable e) {
            if (isUnsubscribed() || finished) {
                RxJavaPlugins.getInstance().getErrorHandler().handleError(e);
                return;
            }
            error = e;
            finished = true;
            schedule();
        }

        protected void schedule() {
            if (counter.getAndIncrement() == 0) {
                recursiveScheduler.schedule(this);
            }
        }

        // only execute this from schedule()
        @Override
        public void call() {
            long emitted = 0L;

            long missed = 1L;

            // these are accessed in a tight loop around atomics so
            // loading them into local variables avoids the mandatory re-reading
            // of the constant fields
            final Queue<Object> q = this.queue;
            final Subscriber<? super T> localChild = this.child;
            final NotificationLite<T> localOn = this.on;
            
            // requested and counter are not included to avoid JIT issues with register spilling
            // and their access is is amortized because they are part of the outer loop which runs
            // less frequently (usually after each RxRingBuffer.SIZE elements)
            
            for (;;) {
                long requestAmount = requested.get();
                long currentEmission = 0L;
                
                while (requestAmount != currentEmission) {
                    boolean done = finished;
                    Object v = q.poll();
                    boolean empty = v == null;
                    
                    if (checkTerminated(done, empty, localChild, q)) {
                        return;
                    }
                    
                    if (empty) {
                        break;
                    }
                    
                    localChild.onNext(localOn.getValue(v));

                    currentEmission++;
                    emitted++;
                }
                
                if (requestAmount == currentEmission) {
                    if (checkTerminated(finished, q.isEmpty(), localChild, q)) {
                        return;
                    }
                }
                
                if (currentEmission != 0L) {
                    BackpressureUtils.produced(requested, currentEmission);
                }
                
                missed = counter.addAndGet(-missed);
                if (missed == 0L) {
                    break;
                }
            }
            
            if (emitted != 0L) {
                request(emitted);
            }
        }
        
        boolean checkTerminated(boolean done, boolean isEmpty, Subscriber<? super T> a, Queue<Object> q) {
            if (a.isUnsubscribed()) {
                q.clear();
                return true;
            }
            
            if (done) {
                if (delayError) {
                    if (isEmpty) {
                        Throwable e = error;
                        try {
                            if (e != null) {
                                a.onError(e);
                            } else {
                                a.onCompleted();
                            }
                        } finally {
                            recursiveScheduler.unsubscribe();
                        }
                    }
                } else {
                    Throwable e = error;
                    if (e != null) {
                        q.clear();
                        try {
                            a.onError(e);
                        } finally {
                            recursiveScheduler.unsubscribe();
                        }
                        return true;
                    } else
                    if (isEmpty) {
                        try {
                            a.onCompleted();
                        } finally {
                            recursiveScheduler.unsubscribe();
                        }
                        return true;
                    }
                }
                    
            }
            
            return false;
        }
    }
}
```

这里的代码有点长，我们先注意到它是一个`Operator`，它没有对上层`Observable`做任何的控制或者包装。

既然是`Operator`，那么它的职责就是把一个`Subscriber`转换成另外一个`Subscriber`， 我们来关注下转换后的`Subscriber`对转换前的`Subscriber`做了些什么事。

首先它是一个`ObserveOnSubscriber`类， 既然是`Subscriber`那么肯定有`onNext`, `onComplete` 和`onError` 看最主要的onNext

```
@Override
public void onNext(final T t) {
    if (isUnsubscribed() || finished) {
        return;
    }
    if (!queue.offer(on.next(t))) {
        onError(new MissingBackpressureException());
        return;
    }
    schedule();
}
```

好了，这里做了两件事，首先把结果缓存到一个队列里，然后调用`schedule`启动传入的`worker`

我们这里需要注意下：

> 在调用`observeOn`前的序列，把结果传入到`onNext`就是它的工作，它并不关心后续的流程，所以工作就到这里就结束了，剩下的交给`ObserveOnSubscriber`继续。

```
protected void schedule() {
    if (counter.getAndIncrement() == 0) {
        recursiveScheduler.schedule(this);
    }
}
```
`recursiveScheduler` 就是之前我们传入的Scheduler，我们一般会在`observeOn`传入`AndroidScheluders.mainThread()`对吧、

接下去，我们看下在`scheduler`中调用的`call`方法，这里只列出主要带代码
```
@Override
public void call() {
    ...
    final Subscriber<? super T> localChild = this.child;
    for (;;) {
        ...
        boolean done = finished;
        Object v = q.poll();
        boolean empty = v == null;
        
        if (checkTerminated(done, empty, localChild, q)) {
            return;
        }
        
        if (empty) {
            break;
        }
        
        localChild.onNext(localOn.getValue(v));

        ...
    }
    
    if (emitted != 0L) {
        request(emitted);
    }
}
```

OK，在`Scheduler`启动后， 我们在`Observable.subscribe(a)`传入的`a`就是这里的`child`， **我们看到，在`call`中终于调用了它的`onNext`方法，把真正的结果传了出去，但是在这里，我们是工作在`observeOn`的线程上的。**

那么总结起来的结论就是：

> 1. `observeOn` 对调用之前的序列默不关心，也不会要求之前的序列运行在指定的线程上
> 2. `observeOn` 对之前的序列产生的结果先缓存起来，然后再在指定的线程上，推送给最终的`subscriber`

## 复杂情况
我们经常多次使用`subscribeOn`切换线程，那么以后是否可以组合`observeOn`和`subscribeOn`达到自由切换的目的呢？

组合是可以的，但是他们的执行顺序是有条件的，如果仔细分析的话，可以知道`observeOn`调用之后，再调用`subscribeOn`是无效的，原因是什么？

因为`subscribeOn`改变的是`subscribe`这句调用所在的线程，大多数情况，产生内容和消费内容是在同一线程的，所以改变了产生内容所在的线程，就改变了消费内容所在的线程。

经过上面的阐述，我们知道，`observeOn`的工作原理是把消费结果先缓存，再切换到新线程上让原始消费者消费，它和生产者是没有一点关系的，就算`subscribeOn`调用了，也只是改变`observeOn`这个消费者所在的线程，和`OperatorObserveOn`中存储的原始消费者一点关系都没有，它还是由`observeOn`控制。



## 总结
如果我们有一段这样的序列

```
Observable
.map                    // 操作1
.flatMap                // 操作2
.subscribeOn(io)
.map                    //操作3
.flatMap                //操作4
.observeOn(main)
.map                    //操作5
.flatMap                //操作6
.subscribeOn(io)        //!!特别注意
.subscribe(handleData)
```

假设这里我们是在主线程上调用这段代码，
那么  
> 1. `操作1`，`操作2`是在io线程上，因为之后`subscribeOn`切换了线程
> 2. `操作3`，`操作4`也是在io线程上，因为在`subscribeOn`切换了线程之后，并没有发生改变。
> 3. `操作5`，`操作6`是在main线程上，因为在他们之前的`observeOn`切换了线程。
> 4. 特别注意那一段，对于`操作5`和`操作6`是无效的
再简单点总结就是
1. `subscribeOn`的调用切换之前的线程。
2. `observeOn`的调用切换之后的线程。
3. `observeOn`之后，不可再调用`subscribeOn` 切换线程

=========
续 特别感谢[@扔物线](http://weibo.com/rengwuxian?is_all=1)给的额外的总结

> 0. 下面提到的“操作”包括产生事件、用操作符操作事件以及最终的通过 subscriber 消费事件
> 1. 只有第一subscribeOn() 起作用（所以多个 subscribeOn() 毛意义）
> 2. 这个 subscribeOn() 控制从流程开始的第一个操作，直到遇到第一个 observeOn()
> 3. observeOn() 可以使用多次，每个 observeOn() 将导致一次线程切换()，这次切换开始于这次 observeOn() 的下一个操作
> 4. 不论是 subscribeOn() 还是 observeOn()，每次线程切换如果不受到下一个 observeOn() 的干预，线程将不再改变，不会自动切换到其他线程
