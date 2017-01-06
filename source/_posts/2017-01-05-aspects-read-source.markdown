---
title: Aspects AOP 的实现
tags:
  - ios
categories:
  - development
date: 2017-01-05 15:26:56
description: 此篇文章是分析下 `Aspects` 的实现，以及对 objc runtime Swizzle 的基本介绍
---


都传闻说 OC 的运行时非常NB，今天就来看看非常有名的`Aspects`，源码在这

https://github.com/steipete/Aspects

里面的内容非常简单，其实就2个文件，`Aspect.h`和`Aspect.m`，它使用`Category`为`NSObject`提供了两个额外的方法，API如下：

```Objective-C
/// Adds a block of code before/instead/after the current `selector` for a specific class.
///
/// @param block Aspects replicates the type signature of the method being hooked.
/// The first parameter will be `id<AspectInfo>`, followed by all parameters of the method.
/// These parameters are optional and will be filled to match the block signature.
/// You can even use an empty block, or one that simple gets `id<AspectInfo>`.
///
/// @note Hooking static methods is not supported.
/// @return A token which allows to later deregister the aspect.
+ (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error;

/// Adds a block of code before/instead/after the current `selector` for a specific instance.
- (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error;

/// Deregister an aspect.
/// @return YES if deregistration is successful, otherwise NO.
id<AspectToken> aspect = ...;
[aspect remove];
```

它提供的解决方案就是为一个消息提供一个 before 和 after 的 block 调用，也就是为 OC 提供了 AOP 的能力。
我们知道在 Java 中，实现 AOP 采用的是动态代理的方式，那么在 OC 中的实现，其实就是通过 Swizzle Method 的方式进行啦。

## 一探究竟
看下 Aspects 到底是如何实现这个功能的

```Objective-C
+ (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error {
    return aspect_add((id)self, selector, options, block, error);
}

/// @return A token which allows to later deregister the aspect.
- (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error {
    return aspect_add(self, selector, options, block, error);
}
```

事实上，不管是静态还是动态方式添加，都是使用`aspect_add`这个方法，

```Objective-C
static id aspect_add(id self, SEL selector, AspectOptions options, id block, NSError **error) {
    NSCParameterAssert(self);
    NSCParameterAssert(selector);
    NSCParameterAssert(block);

    __block AspectIdentifier *identifier = nil;

    // ...
    // 省略加锁的 block 和 权限检查

    // 看 aspect_getContainerForObject 源码可知，使用 lazy load 的方式，为 self 生成一个 AspectsContainer
    AspectsContainer *aspectContainer = aspect_getContainerForObject(self, selector);
    identifier = [AspectIdentifier identifierWithSelector:selector object:self options:options block:block error:error];
    if (identifier) {
        [aspectContainer addAspect:identifier withOptions:options];

        // Modify the class to allow message interception.
        aspect_prepareClassAndHookSelector(self, selector, error);
    }
    return identifier;
}
```
好了，这里的大头是`aspect_prepareClassAndHookSelector`

```Objective-C
static void aspect_prepareClassAndHookSelector(NSObject *self, SEL selector, NSError **error) {
    NSCParameterAssert(selector);

    // 下面一行代码，是动态生成了一个子类，然后覆盖了原先的 forwardInvocation 消息，这是这里最 magic 的地方，以下会讲到
    Class klass = aspect_hookClass(self, error);
    Method targetMethod = class_getInstanceMethod(klass, selector);
    IMP targetMethodIMP = method_getImplementation(targetMethod);

    // 检查如果目标方法的实现还不是 _objc_msgForward 或者 _objc_msgForward_stret 的话，就进行 hook
    if (!aspect_isMsgForwardIMP(targetMethodIMP)) {
        // Make a method alias for the existing method implementation, it not already copied.
        const char *typeEncoding = method_getTypeEncoding(targetMethod);

        // 给我们需要被取代的 selector 取一个别名
        SEL aliasSelector = aspect_aliasForSelector(selector);
        if (![klass instancesRespondToSelector:aliasSelector]) {

            // 为类增加一个名字为 aliasSelector， 实现为 selector 的消息
            __unused BOOL addedAlias = class_addMethod(klass, aliasSelector, method_getImplementation(targetMethod), typeEncoding);
            NSCAssert(addedAlias, @"Original implementation for %@ is already copied to %@ on %@", NSStringFromSelector(selector), NSStringFromSelector(aliasSelector), klass);
        }

        // 把原先的 selector 方法的实现指向 _objc_msgForward 或 _objc_msgForward_stret。
        // 在先前调用了 aspect_hookClass 里面，hook 了 forwardInvocation，后文会说明
        // We use forwardInvocation to hook in.
        class_replaceMethod(klass, selector, aspect_getMsgForwardIMP(self, selector), typeEncoding);
        AspectLog(@"Aspects: Installed hook for -[%@ %@].", klass, NSStringFromSelector(selector));
    }
}
```

好了，经过这么多步骤后，我们理清一下思路，如果我们要对`@selector(viewDidLoad:)`进行 hook

1. 先创建 subclass, hook @selector(forwardInvocation:)，并把当前的对象设置为该类的对象，这样在不污染原类的情况下，实现了`forwardInvocation`的hook
2. 为我们的 obj 的目标消息创建一个别名，这里如果是`viewDidLoad:`的话，那别名就是`aspects_viewDidLoad:`
3. 取代目标消息的实现，使用`aspect_getMsgForwardIMP`来选择是 _objc_msgForward 或是 _objc_msgForward_stret

这样就完成了前置工作，接下来我们简单来讲什么是 _objc_msgForward

## Objc 消息转发简介

我们这篇，主要是讲 Aspects 提供的解决方案，所以不会展开阐述 objc runtime 的一些内容，所以先提供参考资料：

http://blog.ibireme.com/2013/11/26/objective-c-messaging/

objc 中发送消息的方式是主要是在 C 层面调用 obj_msgSend 方法，如果找不到消息的实现，它会尝试进行转发，原理是把函数的实现改为 _objc_msgForward，它是一个函数指针

在转发过程中，objc 会把方法签名包装成 Invocation 传入到 `forwardInvocation:` 里，以下是对博文的引用：

> + Test NSObject initialize
> + Test NSObject new
> + Test NSObject alloc
> + Test NSObject allocWithZone:
> - Test NSObject init
> - Test NSObject performSelector:
> + Test NSObject resolveInstanceMethod:
> - Test NSObject forwardingTargetForSelector:
> - Test NSObject methodSignatureForSelector:
> - Test NSObject class
> - Test NSObject doesNotRecognizeSelector:

> 结合NSObject文档可以知道，_objc_msgForward 消息转发做了如下几件事：
> 1.调用resolveInstanceMethod:方法，允许用户在此时为该Class动态添加实现。如果有实现了，则调用并返回。如果仍没实现，继续下面的动作。
> 2.调用forwardingTargetForSelector:方法，尝试找到一个能响应该消息的对象。如果获取到，则直接转发给它。如果返回了nil，继续下面的动作。
> 3.调用methodSignatureForSelector:方法，尝试获得一个方法签名。如果获取不到，则直接调用doesNotRecognizeSelector抛出异常。
> 4.调用forwardInvocation:方法，将地3步获取到的方法签名包装成Invocation传入，如何处理就在这里面了。
> 上面这4个方法均是模板方法，开发者可以override，由runtime来调用。最常见的实现消息转发，就是重写方法3和4，吞掉一个消息或者代理给其他对象都是没问题的。

经过以上，我们知道了，如果方法的实现是`_objc_msgForward`的话，那我们的消息就会被包装成`Invocation`发送到`forwardInvocation`里去，那么在前面，我们进行`subclass`的时候，就会`forwardInvocation`进行了`hook`，这时候就用到了！
看看 Aspects 是如何做的吧。

## 偷梁换柱

具体函数在`aspect_swizzleForwardInvocation`中实现

```Objective-C
static NSString *const AspectsForwardInvocationSelectorName = @"__aspects_forwardInvocation:";
static void aspect_swizzleForwardInvocation(Class klass) {
    NSCParameterAssert(klass);
    // If there is no method, replace will act like class_addMethod.
    IMP originalImplementation = class_replaceMethod(klass, @selector(forwardInvocation:), (IMP)__ASPECTS_ARE_BEING_CALLED__, "v@:@");
    if (originalImplementation) {
        class_addMethod(klass, NSSelectorFromString(AspectsForwardInvocationSelectorName), originalImplementation, "v@:@");
    }
    AspectLog(@"Aspects: %@ is now aspect aware.", NSStringFromClass(klass));
}
```

我们看到，`Aspects`把`forwardInvocation`的实现换成了`__ASPECTS_ARE_BEING_CALLED__`这个函数，而原始的`forwardInvocation`实现的名字就变成了`__aspects_forwardInvocation`
看看`__ASPECTS_ARE_BEING_CALLED__`这里干了什么

```Objective-C
// This is the swizzled forwardInvocation: method.
static void __ASPECTS_ARE_BEING_CALLED__(__unsafe_unretained NSObject *self, SEL selector, NSInvocation *invocation) {
    NSCParameterAssert(self);
    NSCParameterAssert(invocation);

    // 得到原始 selector
    SEL originalSelector = invocation.selector;
    // 得到原始 selector 的别名（之前被我们添加到类里了），这才是真正的实现
    SEL aliasSelector = aspect_aliasForSelector(invocation.selector);
    // 替换 selector
    invocation.selector = aliasSelector;
    AspectsContainer *objectContainer = objc_getAssociatedObject(self, aliasSelector);
    AspectsContainer *classContainer = aspect_getContainerForClass(object_getClass(self), aliasSelector);
    AspectInfo *info = [[AspectInfo alloc] initWithInstance:self invocation:invocation];
    NSArray *aspectsToRemove = nil;

    // Before hooks.
    aspect_invoke(classContainer.beforeAspects, info);
    aspect_invoke(objectContainer.beforeAspects, info);

    // Instead hooks.
    BOOL respondsToAlias = YES;
    if (objectContainer.insteadAspects.count || classContainer.insteadAspects.count) {
        aspect_invoke(classContainer.insteadAspects, info);
        aspect_invoke(objectContainer.insteadAspects, info);
    }else {
        Class klass = object_getClass(invocation.target);
        do {
            if ((respondsToAlias = [klass instancesRespondToSelector:aliasSelector])) {
                [invocation invoke];
                break;
            }
        }while (!respondsToAlias && (klass = class_getSuperclass(klass)));
    }

    // After hooks.
    aspect_invoke(classContainer.afterAspects, info);
    aspect_invoke(objectContainer.afterAspects, info);

    // 以上就是执行 hook 的具体内容了

    // If no hooks are installed, call original implementation (usually to throw an exception)
    if (!respondsToAlias) {
        // 如果没有执行的话。。那么只好执行默认的 forwardInvocation 了
        invocation.selector = originalSelector;
        SEL originalForwardInvocationSEL = NSSelectorFromString(AspectsForwardInvocationSelectorName);
        if ([self respondsToSelector:originalForwardInvocationSEL]) {
            ((void( *)(id, SEL, NSInvocation *))objc_msgSend)(self, originalForwardInvocationSEL, invocation);
        }else {
            [self doesNotRecognizeSelector:invocation.selector];
        }
    }

    // 做一些额外的清理
    // Remove any hooks that are queued for deregistration.
    [aspectsToRemove makeObjectsPerformSelector:@selector(remove)];
}
```

以上就是`Aspects` hook objc 进行 AOP 全过程，虽然只有短短一千行不到的代码，却提供了很方便的方式进行 AOP，实现也很巧妙。
看完后对 objc runtime 羡慕不已，实在强大！