---
title: 两张图看懂http和https
date: 2019-01-24
tags: ["http","https"]
categories: ["计算机网路"]
---
# http和https
http访问过程

![](https://upload-images.jianshu.io/upload_images/422094-4649bc330f6270db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

https访问过程

![image.png](https://upload-images.jianshu.io/upload_images/422094-8e74d012bfef0dac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

几点说明：
> 服务器端返回证书和公开密钥，公开密钥作为证书的一部分而存在

# Connection: keep-alive
http请求头中，有一个字段`Connection: keep-alive`

若connection 模式为close，则服务器主动关闭TCP 连接，客户端被动关闭连接，释放TCP 连接;

若connection 模式为keepalive，则该连接会保持一段时间，在该时间内可以继续接收请求;

大远的话：一个连接必须两端配合，要不要保持连接必须两边商量着来。
不能服务端已经关闭连接了，客户端还请求。

那怎么商量呢？就用这个字段了`Connection`

# 有keepalive了，还要websocket干什么？

websocket是全双工通信，意味着服务端和客户端是平等的，都可以给对方发信息

http中keepalive只能保持一段时间，而且服务端是不能主动给客户端发信息的。