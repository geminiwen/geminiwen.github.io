---
layout: post
title:  "CrossWalk - android 动态加载so库文件实践"
date:   2015-06-29 18:00:00 +0800
categories: android
---


之前看到`简书`Android客户端使用的编辑器，甚是喜欢，它的优雅以及高性能的特点让我爱不释手，很想自己也去做一个。
此前实现过一个在Android上的[Markdown编辑器](https://github.com/geminiwen/gm-mkdroid)
但是界面以及`所见即所得`的效果非常不好看，所以一直耿耿于怀。

然后冒昧看了下`简书`的布局系统，看见了几个奇怪的类，包括类似`XWalkContentView`，于是Google了下，就查到了`CrossWalk`这个`hybrid`框架了。第一眼并不觉得它有啥不一样，以为是一个`Cordova`的轮子。后来细看，发现是自个儿编辑了整个`Chrominum`，屌屌屌!

运行个demo，wrapper了一个http://sf.gg 发现体验真的是不错啊，webview性能到这个水平内心都宽慰了，但是为何安装速度那么慢呢？一看apk大小，足足有`40M+`，感觉天都要塌了。`SegmentFault for Android` 客户端才`3.03M`，我要是包上这玩意，估计就没多少人下了吧。。。然后又看看`简书`，整个apk大小才8M，在启动编辑器的时候，提示需要下载编辑器，下载了一会，然后再打开。顿时就明白了，看来它的库是从外部载入的，记得以前看到过从外部加载`动态链接库`想想很是简单，于是入坑了。

好嘛，我把so文件先不放进apk中，让apk装好之后，放入`/data/data/<app>/lib`目录下，启动app，直接crash。
看日志入下:

>DavlikDexClassLoader Unsatisfied Link library['/xxxx/xxx.apk', '/vendor/lib', '/system/lib']

一看这个路径，泪奔了，原来`library path`只有三个路径下去检查，算了，我们不是有`System.load`和`System.loadLibrary`函数么，直接调用呗，于是我就先暂时把绝对路径给写了下来，直接调用`System.load`函数。

再次启动，发现`CrossWalk`报`Shared Library should use SharedXWalkView`。但是使用`SharedXWalkView`有许多的限制，比如需要安装一个`CrossWalk Runtime`的apk，奇怪了，它怎么知道我是用`Shared Library`的呢？而且简书也没有说要安装apk啊。

于是我继续研究，开始看`CrossWalk`的源码，找到`ReflectionHelper`这个类里面有一行代码`shouldUseLibrary()`，它会去调用`System.loadLibrary()`如果没有报异常，则返回`false`，否则返回`true`。

我们知道`System.loadLibrary`这个函数，会去`java.library.path`这个环境变量的路径下面寻找库，而`Android`是不允许我们更改这个环境变量的值的，就导致`CrossWalk`认为并没有加载它的`runtime`而去开启`Shared`模式。

OK，知道怎么解决就方便了，首先，我们要把`so`文件放入到`/data/data/<app>/`下的任意路径，因为我们的apk有这个权限在这里放东西，然后使用`System.load`加载这个`so`库，最后使用反射的方式欺骗`CrossWalk`框架，告诉它我们的类库已经加载完毕。

我们仔细研究下它的源码，发现有几个标志位需要更改，具体代码如下：
```
System.load(libPath);
try {
    LibraryLoader loader = LibraryLoader.get(1);
    Class c = Class.forName("org.xwalk.core.internal.XWalkViewDelegate");
    Field field = c.getDeclaredField("sLibraryLoaded");
    field.setAccessible(true);
    field.setBoolean(null, true);
    field.setAccessible(false);

    field = LibraryLoader.class.getDeclaredField("mLoaded");
    field.setAccessible(true);
    field.setBoolean(loader, true);
    field.setAccessible(false);
    PathUtils.setPrivateDataDirectorySuffix("xwalkcore");

} catch (NoSuchFieldException e) {
    e.printStackTrace();
} catch (IllegalAccessException e) {
    e.printStackTrace();
} catch (ClassNotFoundException e) {
    e.printStackTrace();
} catch (ProcessInitException e) {
    e.printStackTrace();
}
```

只要把以上的类中的标志位更改掉，那么`CrossWalk`就认为库已经加载成功了。
