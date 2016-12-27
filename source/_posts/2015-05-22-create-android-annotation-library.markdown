---
layout: post
title:  "打造一个 Android 的注解库"
date:   2015-05-22 18:00:00 +0800
categories: development
tags: android
---

注：本文适合有一定java基础的童鞋看，至少明白注解Annotation是什么
<!-- more -->

> 贴上我的Android网络通信库地址
https://github.com/MyLifeForTheOrc/gm-httpengine-studio

最近在`annotation`分支上工作，就为了增加注解支持。
目标是像[ButterKnife](https://github.com/JakeWharton/butterknife)一样酷炫，现在也差不多。

首先看下改进后的（酷炫）使用方法，如果我需要做一个http请求，只需要以下几步：

### 定义API

```java
package org.gemini.httpengine.examples;

import org.gemini.httpengine.annotation.GET;
import org.gemini.httpengine.annotation.Path;
import org.gemini.httpengine.annotation.TaskId;
import org.gemini.httpengine.library.OnResponseListener;

/**
 * Created by geminiwen on 15/5/21.
 */
public interface UserAPI {
    interface TASKID {
        String TASK_GET_LOGIN = "login";
    }

    @Path("http://www.baidu.com")     //定义URL地址
    @TaskId(TASKID.TASK_GET_LOGIN)    //给这个请求加一个taskId，标识请求
    @GET                              //标明这个请求是一个GET请求
    void login(OnResponseListener l,
               String username,
               String password);
}
```

### 在Activity中调用API
```java
    @Override
    public void onClick(View v) {
        if(v == mTestButton) {
            UserAPI api = InjectFactory.inject(UserAPI.class); //注入API实例
            api.login(this, "geminiwen", "password");   //调用接口
        }
    }
```

### 获取接口
首先实现`OnResponseListener`接口
```java
    @Override
    public void onResponse(GMHttpResponse response, GMHttpRequest request) {
        byte[] result = null;
        try {
            result = response.getRawData();        //获取数据
        } catch (Exception e) {
            Log.e("error", "wtf?", e);
        }

//        Toast.makeText(this,result,Toast.LENGTH_LONG).show();
    }
```

使用了注解的方式，是不是感觉很干净彻底？
但是这里只有接口，并没有实现类啊，这到底是怎么做到的呢？

OK，我们看看它背后做了什么。

这里我们使用`Android Studio`举例

首先可以看下Android Application Module下面的`build`文件夹

![clipboard.png](https://segmentfault.com/img/bVlQLE)
其他文件都很正常，除了两个"不速之客"，`UserAPI$$APIINJECTOR.java` 和 `UserAPI$$APIINJECTOR.class`。没错，这就是使用`Annotation Processor`生成的java文件了。
我们看看生成了什么。

![clipboard.png](https://segmentfault.com/img/bVlQLP)

把它格式化一下如下：
```java
package org.gemini.httpengine.examples;

import org.gemini.httpengine.library.*;

public class UserAPI$$APIINJECTOR implements org.gemini.httpengine.examples.UserAPI {
    
    public void login(org.gemini.httpengine.library.OnResponseListener l, java.lang.String username, java.lang.String password) {
        final String FIELD_USERNAME = "username";
        final String FIELD_PASSWORD = "password";
        GMHttpParameters httpParameter = new GMHttpParameters();
        httpParameter.setParameter(FIELD_USERNAME, username);
        httpParameter.setParameter(FIELD_PASSWORD, password);
        GMHttpRequest.Builder builder = new GMHttpRequest.Builder();
        builder.setHttpParameters(httpParameter);
        builder.setTaskId("login");
        builder.setUrl("http://www.baidu.com");
        builder.setMethod("GET");
        builder.setOnResponseListener(l);
        GMHttpService service = GMHttpService.getInstance();
        service.executeHttpMethod(builder.build());

    }
}
```
这里的代码就是调用`GMHttpEngine`里面带的API，进行HTTP的请求。

OK，我们得到结论了，它的秘密就是库利用一些特殊的特性，帮你生成了一个实现类，并利用`InjectFactory.inject`这个方法，把这个实现类用反射的方式，生成出来，返回给你的接口。

那新问题产生了，`生成代码`这么叼的事是怎么做到的呢。我们必须了解`apt`这个东西的存在
给个APT介绍的传送门（鸟语）：http://docs.oracle.com/javase/7/docs/technotes/guides/apt/

这时候我们知道了有`Annotation Processor`这个东西的存在，我们的目标就是自定义一个`Annotation Processor`了。

看第一个接口叫`AbstractProcessor`，完全限定名是：`javax.annotation.processing.AbstractProcessor;`
它是一个抽象类，所以我们要自定义一个类，去继承它，它最主要的就是里面的`process`接口
```java
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        Map<TypeElement, APIClassInjector> targetClassMap = findAndParseTargets(roundEnv);

        for (Map.Entry<TypeElement, APIClassInjector> entry : targetClassMap.entrySet()) {
            TypeElement typeElement = entry.getKey();
            APIClassInjector injector = entry.getValue();
            try {
                String value = injector.brewJava();

                JavaFileObject jfo = filer.createSourceFile(injector.getFqcn(), typeElement);
                Writer writer = jfo.openWriter();
                writer.write(value);
                writer.flush();
                writer.close();
            } catch (Exception e) {
                processingEnv.getMessager().printMessage(Diagnostic.Kind.ERROR, e.getMessage(), typeElement);
            }
        }


        return false;
    }
```
看见里面一个叫`JavaFileObject`的对象没有，它就是生成java文件最重要的对象了。这里我们干的事情，就是分析我们的`注解(Annotation)`然后，生成相应的代码，利用JavaFileObject进行写入，就好了。

当然，你需要告诉`Annotation Processor`你要处理哪一些注解，具体方法就是重写它的`getSupportedAnnotationTypes`方法
```java
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> supportTypes = new LinkedHashSet<>();
        supportTypes.add(Path.class.getCanonicalName());
        return supportTypes;
    }
```
因为`Path`注解是`GMHttpEngine`所使用的核心注解，所以我这里就写了这个，从接口看，它可以支持一批的注解，对于不同的注解要有不同的处理方式。

除此之外，你需要定义`resources`文件夹，新建一个文件叫`javax.annotation.processing.Processor`
里面的内容就是你的`Annotation Processor`的完全限定名。目录结构如下：

![clipboard.png](https://segmentfault.com/img/bVlQNB)

里面的内容是


![clipboard.png](https://segmentfault.com/img/bVlQNC)


最后说了这么多，再整理下神奇的流程。

1. 首先我们要定义一下自己的注解。
2. 自定义我们自己的`Annotation Processor`，即继承`AbstractProcessor`类。
3. 在`process`方法中分析注解，生成java代码。
4. 在`resources\META-INF\services\javax.annotation.processing.Processor`文件中注册类名。
5. 导入就可以使用注解标识API请求自动生成代码啦~

噢~这里要说下在开发注解功能碰到的坑：

> 如果使用AndroidStudio 需要注意的是，`Android Library`并不是普通的`JavaSE`，所以并没有提供`javax`的一些功能，所以，在新建Module的时候不能选`Android Library`而应该选`Java Library`，而且因为它只在编译的时候使用到`JavaSE`的功能，所以并不用担心在手机上跑的时候会出问题。

好了，想了解更多，欢迎`star` 欢迎`fork`
> https://github.com/MyLifeForTheOrc/gm-httpengine-studio

欢迎在`SegmentFault`上一起讨论问题~

也欢迎给我邮件 [geminiwen@segmentfault.com](mailto:geminiwen@segmentfault.com)