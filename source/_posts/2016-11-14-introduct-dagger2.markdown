---
layout: post
title:  "Dagger2 —— 匪夷所思，结果那么爱你"
date:   2016-11-14 18:00:00 +0800
categories: development
tags: android
photos:
    - /2016/11/14/introduct-dagger2/dagger2.png
---

今天我们来讲讲一个有一点点冷门的库`Dagger`吧。我做一个不负责任的猜测：做客户端的同学可能比较少听到一些名
词，比如`面向切面编程`、`控制反转`、`依赖注入`，相信玩过`Spring`的同学肯定知道这些一开始让人头大后来却很好玩的玩意儿。
<!-- more -->


今天我们来介绍这款依赖注入器 —— `Dagger2`，源自Square的`Dagger`，由Google开发，基于apt生成静态编译时的依赖注入工具，比动态注入的方式更加高性能，但是需要更多的约定。

官网：https://google.github.io/dagger/

# 组成
`Dagger2`（以下称为`Dagger`) 主要由两个部分组成：`Component`和`Module`。分别作为注入器和注入源存在于整个依赖图中，然后有了`源`和`工具`，那么只用在我们需要注入的地方加上`@Inject`注解即可，它是属于`JSR-330`的一部分，我们这里就直接引入一个最简单的Demo。

## Module
```
@Module
public class AppModule {
    Context mApplicationContext;

    public AppModule(Context context) {
        this.mApplicationContext = context;
    }

    @Provides
    public Context provideContext(){
        return mApplicationContext;
    }

    @Provides
    public Service provideService(Context context) {
        return new Service(context, null);
    }
}
```
这是世界的起源。

- `@Module`注解表示这个类是个Module，是一个“源”。
- `@Provides`注解告诉Dagger我们想要构造对象并提供这些依赖。

## Component
```
@Component(modules=AppModule.class)
public interface AppComponent {
    void inject(App app);

    Context context();
    Service service();
}
```
`Component`是一个接口，具体的实现由`Dagger`经过`apt`工具为你生成（是不是有了apt就特别爽），我们给`AppComponent`这枚“针”指定了药剂——`AppModule`，告诉它注入的时候，从`AppModule`里面拿到我们要的变量实例，只要给`Component`声明一个无返回值，带`被注入类型形参`的方法，Dagger 就会为这个类生成一个`MemberInjector`对象，用来给被注入类注入对象。

## 被注入对象

```
public class App extends Application {

    @Inject Service mService;
    
    private AppComponent mAppComponent;

    /**
     * 应用程序初始化
     */
    @Override
    public void onCreate() {
        super.onCreate();
        mAppComponent = DaggerAppComponent.builder()
                .appModule(new AppModule(this))
                .build();

        mAppComponent.inject(this);

    }
    
    public AppComponent getAppComponent() {
        return mAppComponent;
    }
}
```
这里的`DaggerAppComponent`是`Dagger`生成的，它的实现类会在所有`Component`接口类之前增加一个`Dagger`前缀，我们只用传入它所需要的依赖即可，`Component`显然是依赖`Module`的，所以需要在这里传入`AppModule`，现在，只要用`Context`的地方，我们都可以拿到这个`AppComponent`实例，只要有这个实例，我们可以在任意地方注入被管理的类。

## 作用域

我们说依赖注入的时候，作用域（Scope）是经常会出现在我们眼里的词汇。控制变量生命周期，实质就是控制它存在的作用域，服务端典型的作用域如`单例(Singleton)`，`Request`，`Session`，等等，它们的变量分别存在于不同的生命周期。

我们默认存在的是`Singleton`，也就是`@Singleton`注解。由它标注的`Provider`生成的对象会被缓存起来，用`SingleCheck`或者`DoubleCheck`进行包装。我们`Provider`指定的作用域需要和`Component`的作用域一致。

比如`Component`这样定义：
```
@Singleton
@Component(modules=AppModule.class)
public interface AppComponent {
    void inject(App app);

    Service service();
}
```
而`Module`就是这个样子
```
@Module
public class AppModule {
    Context mApplicationContext;

    public AppModule(Context context) {
        this.mApplicationContext = context;
    }

    @Singleton
    @Provides
    public Context provideService(){
        return new Service();
    }

}
```

## 限定符
`Dagger`还支持使用`限定符（Qualifier）`来指定注入的对象，比如内置的`@Named`限定符，我们在需要特定限定名字的变量的时候，可以在`@Inject`上，指定`@Named`限定符，获取指定对象。
```
//Module
@Providers
@Named("cache")
public Service provideService();

// Injection
@Inject
@Named("cache")
Service mService;
```
这样就给这个`mService`注入了名字为"cache"的实例了。

## 一个简单场景的应用

当我们谈论依赖注入的时候，我们在谈论什么？
其实我们是在讨论 **作用域**。

这是什么意思呢，我相信每一个程序员去实现一个单例是一件非常简单的事情，`makeInstance`和`getInstance`嘛。但是，你们想过维护“双例”吗？我之前碰到了一个场景如下：

> 1. 用户(User) 需要一张tag表，会增删查改，并和用户相关联。
> 2. 问题(Question) 也需要一张tag表，也会增删查改，和问题关联。
> 3. 这两张表的实体模型一模一样。

那么我们采取的方案有两种，双表，或者双库。双表的话，对ORM很不友好，因为ORM是根据类来确定表的，我们为了代码简洁优雅，不可能创建两个一模一样的类，不然取名都变成一件困难的事。这里，一个优势是，我使用的ORM
库首先是维护一个单例，单例进行CRUD操作，且一个单例和一个数据库相关。于是我使用`Qualifer`的特性，生成了两个实例（也就是对应了两个数据库），分别注入到不同的业务模型中去，他们就可以使用同一个类，而且对tag的修改完全没有影响。
这件事要是我们自己去做的话，可能要写很多肮脏不堪的代码，但是Dagger只用2个注解就把我的需求解决了。

# 总结

好了，本文简单的阐述了Dagger入门使用，总结起来，我们只要约定好`Componenet`、`Module`，搭配`Inject`使用，即可实现一个静态的依赖注入流程。下一次我们详细介绍`Dagger`生成的代码结构。