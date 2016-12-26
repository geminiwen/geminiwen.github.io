---
layout: post
title:  "Android 实现锁屏的较完美方案"
date:   2015-08-11 18:00:00 +0800
categories: android
---

实现锁屏的方法，其实上网一搜一大把，无非是告诉你在`Screen Off`的时候启动某个Activity，同时把`Keyguard`禁用掉，但是通常情况下，似乎很难解决`HOME`键能解锁的这么一个问题，今天我们就来讲一个方案，就是如何近乎完美的实现我们的锁屏。

我们知道，锁屏的界面显示是使用`KeyguardViewManager`进行添加的，但是这个类属于Android的内部类，我们调用不到，它属于`com.android.internal.policy.impl`这个包，源码地址：https://github.com/android/platform_frameworks_policies_base/blob/master/phone/com/android/internal/policy/impl/KeyguardViewManager.java

我们可以看到它似乎是使用WindowManager添加View的方式实现了锁屏界面的添加，而不是使用传统的Activity的方式。

那么使用`WindowManager`是我们可行的方案，它的好处就是可以屏蔽Home键的触发，同时可以做一些特殊的动画效果。

我们首先开启一个`Service`，在`Service`中注册`SREEN_OFF`的广播，触发广播的时候，使用WindowManager加入锁屏页面，实现锁屏即可。
看下我们的Receiver代码：
```java
if (action.equals(Intent.ACTION_SCREEN_OFF)) {
    ViewParent viewParent = mContainer.getParent();
    if (viewParent != null) {
        return;
    }

    mKeyguardLock.disableKeyguard();
    WindowManager windowManager = (WindowManager)getSystemService(WINDOW_SERVICE);
    WindowManager.LayoutParams lp = generateLayoutParams();
    windowManager.addView(mContainer, lp);
}


 private WindowManager.LayoutParams generateLayoutParams() {
    WindowManager windowManager = (WindowManager)getSystemService(WINDOW_SERVICE);
    Display display = windowManager.getDefaultDisplay();
    Point size = new Point();
    display.getSize(size);

    WindowManager.LayoutParams lp = new WindowManager.LayoutParams();
    lp.type = WindowManager.LayoutParams.TYPE_SYSTEM_ALERT;
    lp.flags |= WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED | WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN;
    lp.x = 0;
    lp.y = 0;
    lp.format = PixelFormat.TRANSLUCENT;
    return lp;
}
```

这里主要的是需要设置WindowManager的布局参数（LayoutParams），如果看WindowManager的源码的话，我们可以看见它的TYPE有一个`TYPE_KEYGUARD`，这就是系统锁屏用的类型了，但是它不提供给外部使用，因此我们只能使用级别比较高的`TYPE_SYSTEM_ALERT`，覆盖到锁屏的上面。`format`设置成`TRANSLUCENT`的原因是我们需要和锁屏交互的时候，锁屏后面的界面要显示出来，所以它是一个透明的层，这里没有办法，我们只能把一些交互的代码放到需要添加的View层中进行处理。

使用这种方式实现的锁屏，能较好的和`Launcher`或者其他界面交互（包括渐变、过渡等），而且能使得我们的锁屏界面不响应`HOME`键（使用Activity的方式的话，home会让我们进入到`Launcher`里）。

更多的内容我正在探索，敬请期待。
> 欢迎关注我[Github](https://github.com/geminiwen) 以及 [@Gemini](http://weibo.com/coffeesherk/home?leftnav=1) 