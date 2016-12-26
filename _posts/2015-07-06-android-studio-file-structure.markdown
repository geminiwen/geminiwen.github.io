---
layout: post
title:  "Android Studio目录结构浅析"
date:   2015-07-06 18:00:00 +0800
categories: android
---

应各位朋友的要求，写这篇文章，让我们来简单了解下Android Studio中不同目录（文件）的位置和用途。
首先看下一个App的最简单的目录结构

![clipboard.png](https://segmentfault.com/img/bVmBbS)
【= = 好复杂的样子】

OK，我们这么看，第一，把这么多文件先分成这么三块
1. 编译系统（Gradle）
2. 配置文件
3. 应用模块

`Gradle`是Google推荐使用的一套基于`Groovy`的编译系统脚本（当然，你也可以使用ant），具体的介绍和文档可以参考这个传送门：https://developer.android.com/tools/building/plugin-for-gradle.html
如果你学会之后，会对`Android`项目的编译了如指掌（总之非常爽~），它的缺点目前是效率不高，然后因为有功夫网的存在，所以在`bintray`上下载依赖会比较慢。

上面那个图中出现`gradle`字眼的就是`gradle`相关的一些文件。
Android中使用`Gradle Wrapper`对`Gradle`进行了一层包装，我猜测这么做的原因是因为gradle更新速度实在太快，为了兼容性着想，才出了这么一套方案。（如果觉得这个猜想有问题请指正）
`gradlew`相关的文件就是和`Gradle Wrapper`有关。我们对除了`app`文件夹以外的文件列一下。

|文件（夹）名|用途|
|---|--|
|.gradle|Gradle编译系统，版本由wrapper指定|
|.idea|Android Studio IDE所需要的文件|
|build|代码编译后生成的文件存放的位置|
|gradle|wrapper的jar和配置文件所在的位置|
|.gitignore|git使用的ignore文件|
|build.gradle|gradle编译的相关配置文件（相当于Makefile）|
|gradle.properties|gradle相关的全局属性设置|
|gradlew|*nix下的`gradle wrapper`可执行文件|
|graldew.bat|windows下的`gradle wrapper`可执行文件|
|local.properties|本地属性设置（key设置，android sdk位置等属性），这个文件是不推荐上传到VCS中去的|
|settings.gradle|和设置相关的gradle脚本|

这些就是外部文件相关的一些文件的介绍。我们来看下更重要的`app`模块里的文件

![clipboard.png](https://segmentfault.com/img/bVmBcE)
这是app模块下的文件目录结构，介绍下他们的用途

|文件（夹）名|用途|
|---|--|
|build|编译后的文件存在的位置（包括最终生成的apk也在这里面）|
|libs|依赖的库所在的位置（`jar`和`aar`)|
|src|源代码所在的目录|
|src/main|主要代码所在位置（src/androidTest)就是测试代码所在位置了|
|src/main/assets|android中附带的一些文件|
|src/main/java|最最重要的，我们的java代码所在的位置|
|src/main/jniLibs|jni的一些动态库所在的默认位置(.so文件)|
|src/main/res|android资源文件所在位置|
|src/main/AndroidManifest.xml|AndroidManifest不用介绍了吧~|
|build.gradle|和这个项目有关的gradle配置，相当于这个项目的Makefile，一些项目的依赖就写在这里面|
|proguard.pro|代码混淆配置文件|

以上就是对`Android Studio`目录结构的简单介绍~

有问题可以直接留言，我会尽快回复。

> 欢迎关注我[Github](https://github.com/geminiwen) 以及 [@Gemini](http://weibo.com/coffeesherk/home?leftnav=1) 