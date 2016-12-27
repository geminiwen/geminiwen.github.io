---
layout: post
title:  "Laravel 的 Session机制简介"
date:   2015-09-28 18:00:00 +0800
categories: development
tags: android
---

前些天，为了解答一个问题，就去研究了Laravel的源码，讲讲我的收获：
这个是问题源：
http://segmentfault.com/q/1010000003776645?_ea=365137
<!-- more -->

本文讲的`Laravel`全部使用`5.1`版本。

我们首先看Laravel是如何创建Session组件的。
首先我们可以看见在`Kernel.php`中注册了`StartSession`这个类（这里不去讨论Laravel的DI和IoC），看下这个类是如何使用的。

## Session Start

我们在这个类中的`handle`方法看到如下代码
```php
public function handle($request, Closure $next)
{
    $this->sessionHandled = true;

    // If a session driver has been configured, we will need to start the session here
    // so that the data is ready for an application. Note that the Laravel sessions
    // do not make use of PHP "native" sessions in any way since they are crappy.
    if ($this->sessionConfigured()) {
        $session = $this->startSession($request);

        $request->setSession($session);
    }

    $response = $next($request);

    // Again, if the session has been configured we will need to close out the session
    // so that the attributes may be persisted to some storage medium. We will also
    // add the session identifier cookie to the application response headers now.
    if ($this->sessionConfigured()) {
        $this->storeCurrentUrl($request, $session);

        $this->collectGarbage($session);

        $this->addCookieToResponse($response, $session);
    }

    return $response;
}
```


OK，一个典型的过滤器，在这个`StartSession`中获取Session的方法是`getSession`。

## Get Session
它使用了`Laravel`注入的`SessionManager` 这个类的完整限定名是：`\Illuminate\Session\SessionManager`
```php
/**
 * Start the session for the given request.
 *
 * @param  \Illuminate\Http\Request  $request
 * @return \Illuminate\Session\SessionInterface
 */
protected function startSession(Request $request)
{
    with($session = $this->getSession($request))->setRequestOnHandler($request);

    $session->start();

    return $session;
}

/**
 * Get the session implementation from the manager.
 *
 * @param  \Illuminate\Http\Request  $request
 * @return \Illuminate\Session\SessionInterface
 */
public function getSession(Request $request)
{
    $session = $this->manager->driver();
    $session->setId($request->cookies->get($session->getName()));

    return $session;
}
```

我们在这个函数中看到，它调用了`SessionManager`的`drive`方法，`drive`方法是`SessionManager`的父类`Manager`中定义的，看到它的实现是这样的

```php
/**
 * Get a driver instance.
 *
 * @param  string  $driver
 * @return mixed
 */
public function driver($driver = null)
{
    $driver = $driver ?: $this->getDefaultDriver();

    // If the given driver has not been created before, we will create the instances
    // here and cache it so we can return it next time very quickly. If there is
    // already a driver created by this name, we'll just return that instance.
    if (! isset($this->drivers[$driver])) {
        $this->drivers[$driver] = $this->createDriver($driver);
    }

    return $this->drivers[$driver];
}

/**
 * Create a new driver instance.
 *
 * @param  string  $driver
 * @return mixed
 *
 * @throws \InvalidArgumentException
 */
protected function createDriver($driver)
{
    $method = 'create'.ucfirst($driver).'Driver';

    // We'll check to see if a creator method exists for the given driver. If not we
    // will check for a custom driver creator, which allows developers to create
    // drivers using their own customized driver creator Closure to create it.
    if (isset($this->customCreators[$driver])) {
        return $this->callCustomCreator($driver);
    } elseif (method_exists($this, $method)) {
        return $this->$method();
    }

    throw new InvalidArgumentException("Driver [$driver] not supported.");
}

```
也就是调用`getDefaultDriver`方法
```php
/**
 * Get the default session driver name.
 *
 * @return string
 */
public function getDefaultDriver()
{
    return $this->app['config']['session.driver'];
}
```

这里就是我们在`app/config/session.php`中定义的`driver`字段，获取这个字段后，我们可以从`createDriver`的源码看到，它实际上是调用`createXXXXDriver`的方式获取到`driver`的。

OK，我们看下最简单的`File`，也就是`createFileDriver`
```php
/**
 * Create an instance of the file session driver.
 *
 * @return \Illuminate\Session\Store
 */
protected function createFileDriver()
{
    return $this->createNativeDriver();
}

/**
 * Create an instance of the file session driver.
 *
 * @return \Illuminate\Session\Store
 */
protected function createNativeDriver()
{
    $path = $this->app['config']['session.files'];

    return $this->buildSession(new FileSessionHandler($this->app['files'], $path));
}

/**
 * Build the session instance.
 *
 * @param  \SessionHandlerInterface  $handler
 * @return \Illuminate\Session\Store
 */
protected function buildSession($handler)
{
    if ($this->app['config']['session.encrypt']) {
        return new EncryptedStore(
            $this->app['config']['session.cookie'], $handler, $this->app['encrypter']
        );
    } else {
        return new Store($this->app['config']['session.cookie'], $handler);
    }
}
```
这个`Store`就是我们`Session`的实体了，它的具体读写调用使用抽象的`Handler`进行，也就是说如果是读文件，就`new FileHandler`如果是Redis就是`new RedisHandler`实现`read`和`write`即可。

## Session ID
回到`StartSession` 这个类，我们再看`getSession`这个方法
```php
 $session->setId($request->cookies->get($session->getName()));
```
了解Web的人都应该知道，session是根据cookie中存的key来区分不同的会话的，所以`sessionId`尤为重要，这里我们先不关心`Cookie`是如何读取，我们知道在这里读取了cookie中存放的SessionId就够了。

接下去看怎么加载数据

## Data
依然是`StartSession`这个类，
我们看`startSession`这个方法，下一句调用的就是`$session->start();`，这句话是干嘛呢？经过前面的分析，我们知道`$session`已经是`Store`对象了，那么它的`start`方法如下：
```php
public function start()
{
    $this->loadSession();

    if (! $this->has('_token')) {
        $this->regenerateToken();
    }

    return $this->started = true;
}

/**
 * Load the session data from the handler.
 *
 * @return void
 */
protected function loadSession()
{
    $this->attributes = array_merge($this->attributes, $this->readFromHandler());

    foreach (array_merge($this->bags, [$this->metaBag]) as $bag) {
        $this->initializeLocalBag($bag);

        $bag->initialize($this->bagData[$bag->getStorageKey()]);
    }
}

 /**
 * Read the session data from the handler.
 *
 * @return array
 */
protected function readFromHandler()
{
    $data = $this->handler->read($this->getId());

    if ($data) {
        $data = @unserialize($this->prepareForUnserialize($data));

        if ($data !== false && $data !== null && is_array($data)) {
            return $data;
        }
    }

    return [];
}
```

依次调用了`loadSession`和`readFromHandler` 好，可以看到从`handler`中调用了`read`方法，这个方法就是从我们之前定义的各种`handler`中读取数据了，它是个抽象方法，在子类中具体实现，然后返回给Store（存入到`Store`中的`attributes`数组中去了，到此为止，`Session`的初始化方法都结束了，
```
public function get($name, $default = null)
{
    return Arr::get($this->attributes, $name, $default);
}
```
这个函数去读取`Session`中的数据了。

## Summary
OK，Session整个初始化的过程总结下：

> `StartSession`#`handle` -> `StartSession`#`startSession` -> `StartSession`#`getSession` -> `SessionManager`#`getDefaultDriver` -> `SessionManager`#`createXXXHandler` -> `Store` -> `Store`#`setId` -> `Store`#`startSession`

`Laravel`巧妙的使用了面向对象的`接口`方式，为我们提供了各种各样不同的存储方式，一旦我们了解了存储方式和加密规则，让不同的web容器进行`Session`共享的目的也可以达到~
