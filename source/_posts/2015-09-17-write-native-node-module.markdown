---
layout: post
title:  "编写Node原生模块"
date:   2015-09-17 18:00:00 +0800
categories: development
tags: node
---


平常我们写`node module`的时候，都是直接用`javascript`去写，今天我们来学习下如何使用`c/c++`来写node模块，用`c/c++`写的优势就在于，你可以调用许多系统级的API，如`fork`，缺点就是它强平台依赖的，不一定能在所有平台下去运行。

> 写一个`node addon`一点都不可怕 * 3

我们用到的工具有2个
> 1.[cmake-js](https://www.npmjs.com/package/cmake-js) 代替node-gyp，使用起来很方便。
> 2.nodejs[源码](https://nodejs.org/dist/latest/)（需要一些头文件和库）

`cmake-js`是使用`CMake`作为工具，构建跨平台的`Makefile`，极大方便了`Makefile`配置的一个js工具。

我们做一个`Hello World`
效果如
```js
module.exports.hello = function() { return 'world'; };
```

废话不多，直接上代码
```c
// hello.cc
#include <node.h>

namespace demo {    //此处的命名空间应该和你的模块名一致

using v8::FunctionCallbackInfo;
using v8::HandleScope;
using v8::Isolate;
using v8::Local;
using v8::Object;
using v8::String;
using v8::Value;

void Method(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();
  args.GetReturnValue().Set(String::NewFromUtf8(isolate, "world"));   //设置返回值
}

void init(Local<Object> exports) {
  NODE_SET_METHOD(exports, "hello", Method);    //注册函数
}

NODE_MODULE(demo, init)    //注册模块

}  // namespace demo
```

比起普通的js代码，的确要复杂很多呀~

然后我们写好`CMakeList.txt`
```
cmake_minimum_required(VERSION 3.3)     #设定cmake的版本 
project(hello)                          #设定项目名称

include_directories(${CMAKE_JS_INC})    #加载由cmake-js提供的环境变量


set(SOURCE_FILES "hello_world.cc")      #设置我们需要编译的文件列表

add_library(hello SHARED exports.cpp ${SOURCE_FILES})  #标明我们要编译成一个lib，使用定义好的文件

set_target_properties(hello PROPERTIES PREFIX "" SUFFIX ".node") #设定编译出来的文件名（默认是libxxx.so，这里改成xxx.node）
```

好，在这个文件夹下面执行`cmake-js`，然后只要编译链接通过，在你的`Build/Release`下面就会出现`hello.node`，

然后在你的js文件里进行测试
```js
var hello = require("hello").hello;
console.log(hello());  //输出 world
```

哈，是不是很简单！
接下去的任务就是好好学习v8相关的API了，在`C/C++`环境下，要千万注意内存泄露的问题！
> 欢迎关注我[Github](https://github.com/geminiwen) 以及 [@Gemini](http://weibo.com/coffeesherk/home?leftnav=1)