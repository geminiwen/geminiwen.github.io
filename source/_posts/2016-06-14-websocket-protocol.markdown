---
layout: post
title:  "websocket 协议解析 [RFC6455]"
date:   2016-06-14 18:00:00 +0800
categories: development
tags: protocol websocket
---

# 序
近期工作忙碌，为了赶`SegmentFault for Android 4.0`版，到了发疯的程度。
我来汇报一个进度，已经实现基于`websocket`的私信系统了，多亏了70大大不懈的努力，在不久的将来，我们使用web给手机发送私信的愿望很快就可以达成了。

那么在这之余，因为个人对各个通信协议都有颇有兴趣，便顺便去看了下`ietf`写的关于`websocket`的文章，原文链接在这：

https://tools.ietf.org/html/rfc6455

当然如果你对实现`websocket`协议并没什么兴趣的话，本文可以直接略过，因为你大概知道怎么用就可以了= =

# 概念与背景
`websocket` 目前仍未有标准化的文档，所以目前我的理解是基于`RFC6455`草案。

`websocket`的诞生场景是因为我们在浏览器上缺少一种与服务器保持长连接的标准化的技术，因为`HTTP 1.1`（以下简称为`HTTP`）协议只是一个标准的无状态协议，并不存在除了`request`和`response`生命周期之外的通信场景。如果我们需要实现在线聊天的功能的话，那么基于`HTTP`，我们只能使用丑陋的`long polling`和`ajax 轮询`等非正常的技术实现我们需要的功能。这些方案虽然解决了我们的使用场景，但并不是最好的方案，我们的`HTTP`服务器软件并不是设计来保持长连接的。

基于以上场景，`websocket`诞生了。
> 1. 它是一种真正的长连接，能让服务端和客户端进行持久的通信，轻松的实现服务器推送技术。
> 2. 它和`HTTP`并没有特别多的关系，但是它的握手 (handshake) 是利用`HTTP`来完成。
> 3. 它的本质是基于`TCP`的连接，可以说和`HTTP`属于平级的应用层协议。

草案阐述了`websocket`的设计目标，除了建立起基于TCP的一层连接以外，还有以下几个特点
> - 为浏览器增加一个基于域名的安全模型（也就是跨域安全模型）
> - 增加一个基于域名和地址的机制，用来支持在一个IP地址上的一端口多域名的多服务模型（类似nginx在同一个ip上使用不同的域名来路由到不同的服务上）
> - 在流式的TCP建立一个帧数据包模型，并且没有长度限制
> - 增加了一个额外的关闭连接握手，给代理和其他中间件使用。

# 连接时握手
`websocket`的握手实际上就是给服务器发送一个`GET`请求，里面带上指定的`header`即可。

`request`例子如下

```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
```

其中比较特殊的是`Upgrade`，`Connection`，和`Sec`开头的几个字段，那么如果请求握手的话，
```
Upgrade: websocket
Connection: Upgrade
```
是固定的要填写的两个键值对。
`Sec-WebSocket-Key`是一个16位的随机值，经过base64编码后生成，给服务器进行UUID连接再编码后由客户端检查用。
`Sec-WebSocket-Version`是使用的版本号。
`Sec-WebSocket-Protocol`是选用的子协议，此字段为可选字段，由服务器选择一个子协议与客户端通信，子协议是由`websocket`承载的协议。

`response`例子如下
```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```
我们可以看到这是一个状态码为`101`的响应，响应的头内容基本和`request`可以对应，`Sec-WebSocket-Accept`是服务端利用`Key`和`UUID`拼接后再进行base64编码产生的一个值，由客户端进行验证。
这样，我们的连接时握手就完成了。

# 数据帧

## 基础帧协议
因为一些安全的原因，从客户端发送到服务端的帧全部要与`掩码`进行异或运算过才有效，而服务端发送到客户端的帧不需要进行异或运算。

我们来看下官方的一幅帧结构定义图

![clipboard.png](https://segmentfault.com/img/bVxZFC)

接下来逐一解释。

|名称|长度|注释|
|--|--|--|
|FIN|1bit|标明这一帧是否是整个消息体的最后一帧|
|RSV1 RSV2 RSV3|1bit|保留位，必须为0，如果不为0，则标记为连接失败|
|opcode|4bit|操作位，定义这一帧的类型|
|Mask|1bit|标明承载的内容是否需要用掩码进行异或|
|Masking-key|0 or 4bytes|掩码异或运算用的key|
|Payload length|7bit or 7 +16bit or 7 + 64bit|承载体的长度（后续会解释为什么会有3种长度）|

如果从结构角度讲，那么`websocket`帧结构就这么简单。

## 操作码 (opcode)

在websocket中，我们定义了几种操作类型，也就是表明了数据包的行为，数据包大体可分为两种，一种是`字符数据包 (string)`，一种是`字节数据包 (byte) `不同的数据包使用不同的`opcode`来传输，`opcode`定义如下：
首先`Opcode`占用`4bit`的长度
|值|定义|
|--|--|
|%x0|标明这一个数据包是上一个数据包的延续，它是一个延长帧 (continuation frame)|
|%x1|标明这个数据包是一个字符帧 (text frame)|
|%x2|标明这个数据包是一个字节帧 (binary frame)|
|%x3-7|保留值，供未来的非控制帧使用|
|%x8|标明这个数据包是用来告诉对方，我方需要关闭连接|
|%x9|标明这个数据包是一个心跳请求 (ping) |
|%xA|标明这个数据包是一个心跳响应 (pong) |
|%xB-F|保留至，供未来的控制帧使用|

## 关于掩码 (Mask)

如果是客户端发送到服务端的数据包，我们需要使用掩码对`payload`的每一个字节进行异或运算，生成`masked payload` 才能被服务器读取。
具体的运算其实很简单。

假设`payload`长度为`pLen`，`mask-key`长度为`mLen`，`i`作为`payload`的游标，`j`作为`mask-key`的游标，伪代码如下：
```java
for (i = 0; i < pLen; i++){
    int j = i % mLen;
    maskedPayload[j] = payload[j] ^ maskKey[j];
}
```


## Payload长度
`Payload Length`位占用了可选的`7bit`或者`7 + 16bit` 或者 `7 + 64bit`，这里是什么意思呢？ MDN上有文章也是对`websocket`协议进行了很好的阐述，先贴原文：

> [编写websocket服务器](https://developer.mozilla.org/zh-CN/docs/WebSockets/Writing_WebSocket_servers)

引用其中关于`Payload length`定义的一段文字：

> ### 读解负载数据长度
> 读取负载数据，需要知道读到那里为止。因此获知负载数据长度很重要。这个过程稍微有点复杂，要以下这些步骤：
> 1. 读取9-15位 (包括9和15位本身)，并转换为无符号整数。如果值小于或等于125，这个值就是长度；如果是 126，请转到步骤 2。如果它是 127，请转到步骤 3。
> 2. 读取接下来的 16 位并转换为无符号整数，并作为长度。
> 3. 读取接下来的 64 位并转换为无符号整数，并作为长度。

当然我们这边所使用的都是`网络字节序`。

## 关闭连接时的握手

关闭连接的时候，只用发送一个`opcode`为0x08的帧，`payload`中前2个字节写入定义的`code`，后续写入关闭连接的`reason`，那么一个关闭流程就握手就开始，此处不再赘述。

# 总结
`websocket`协议整体看下来其实还是很简单，也为我们的工作节省了很多不必要的麻烦。也许我们并不需要了解它实现的细节（除非你正在开发非浏览器的客户端或者服务端）

总而言之，想要学习一个技术，看原始规范果然会让人畅快淋漓啊~

