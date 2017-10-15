---
layout: post
title: AFNetworking 3.1.0 源码学习之概览
date: 2017-10-15 10:54:24.000000000 +09:00
tags: iOS
---

做iOS开发，`AFNetworking`几乎是用来做网络请求的标配。这是一个非常易于使用的框架，但里面具体是怎么构造一个网络请求，怎么接收网络应答，除此之外又做了哪些额外的工作，值得我们去深入学习。并且该框架具有良好的`架构设计`、`编码规范`和详尽的文档，适合作为我们学习开源框架的首选。

本文基于[AFNetworking 3.1.0](https://github.com/AFNetworking/AFNetworking)进行学习，首先看下官方下载下来的源码结构：

![](/assets/images/2017/afnetworking-study-overview-1.png)

`AFNetworking Example`是官方的Demo，而上面的`AFNetworking`则是该框架的源代码。其中`AFNetworking`文件夹就是框架的核心源代码，也是我们学习的重点，而下面的`UKit+AFNetworking`则是对UIKit的扩展。

接下来，我们点开`AFNetworking`文件夹看下：

![](/assets/images/2017/afnetworking-study-overview-2.png)

我们看到`AFNetworking`有六个class，每个class的作用分别如下：

- `AFHTTPSessionManager`：继承并扩展AFURLSessionManager，用于进行HTTP协议的网络通信
- `AFNetworkReachabilityManager`：用于监听网络状态变化的模块
- `AFSecurityPolicy`：用于设置网络通信安全策略的模块
- `AFURLRequestSerialization`：用于网络通信请求信息的序列化/反序列化的模块
- `AFURLResponseSerialization`：用于网络通信应答信息的序列化/反序列化的模块
- `AFURLSessionManager`：核心网络通信模块

这六个模块的结构关系如下：

![](/assets/images/2017/afnetworking-study-overview-3.png)

核心通信模块是`AFURLSessionManager`，`AFHTTPSessionManager`继承自`AFURLSessionManager`，而`AFSecurityPolicy`、`AFNetworkReachabilityManager`、`AFURLRequestSerialization`、`AFURLResponseSerialization`这四个模块则是较为独立模块，为`AFURLSessionManager`和`AFHTTPSessionManager`所调用。这四个模块也说明了`AFNetworking`框架在进行网络通讯时做了其他三件事情，分别是设置网络通信安全策略，监听网络状态变化和请求/应答的信息序列化以及反序列化。

接下来我将会对`AFNetworking`这六个模块进行详细的学习和记录。