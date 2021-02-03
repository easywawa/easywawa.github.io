---
layout: post
category: openstack
title: openstack组件通信方式解析
tagline: by 噜噜噜
tags: 
  - xx
published: true
---



<!--more-->

### 一、组件间通信关系

**OpenStack 组件之间的通信分为四类：**

```
基于 HTTP 协议
基于 AMQP(Advanced Message Queuing Protocol，一个提供统一消息服务的应用层标准高级消息队列协议) 协议（基于消息队列协议）
基于数据库连接（主要是 SQL 的通信）
Native API（基于第三方的 API）
```

#### 1、基于HTTP协议进行通信

```
通过各项目的 API 建立的通信关系，基本上都属于这一类，这些 API 都是 RESTful Web API：
最常见的就是通过 Horizon 或者说命令行接口对各组件进行操作的时候产生的这种通信
然后就是各组件通过 Keystone 对用户身份进行校验，进行验证的时候使用这种通信
还有比如说 Nova Compute 在获取镜像的时候和 Glance 之间，对 Glance API 的调用
```

#### 2、基于高级消息队列协议

```
基于 AMQP 协议进行的通信，主要是每个项目内部各个组件之间的通信
比方说 Nova 的 Nova Compute 和 Scheduler 之间，Cinder 的 Scheduler 和 Cinder Volume之间
```

#### 3、基于SQL通信

```
通过数据库连接实现通信，这些通信大多也属于各个项目内部，之间通过基于 SQL 的这些连接来进行通信。OpenStack 没有规定必须使用哪种数据库，虽然通常用的是 MySQL
```

#### 4、基于Native API通信

```
出现在 OpenStack 各组件和第三方的软硬件之间，比如说，Cinder 和存储后端之间的通信，Neutron 的 agent 或者说插件和网络设备之间的通信，
这些通信都需要调用第三方的设备或第三方软件的 API，我们称为它们为 Native API，那么这个就是我们前面说的基于第三方 API 的通信。
```

### 二、openstack API

OpenStack 的逻辑关系是要各个组件之间的信息传输来实现的，而组件之间的信息传输主要是通过OpenStack 之间相互调用 API 来实现的，作为一个操作系统，作为一个框架，它的 API 有着重要的意义

#### 1、RESTful Web API

目前最常见的实现方式就是**基于 HTTP 协议**实现的 RESTful Web API，我们的 OpenStack 里面用的就是这种方式。REST 架构里面对资源的操作，包括：获取、创建、修改和删除，正好对应着 HTTP 协议提供的 GET、POST、PUT 和 DELETE 方法，所以用 HTTP 来实现 REST 是比较方便的

#### 2、特点

RESTful Web API 主要有以下三个要点：

```
资源地址与资源的 URI，比如：http://example.com/resources/
传输资源的表现形式，指的是 Web 服务接受与返回的互联网媒体类型，比如：JSON，XML 等，其中 JSON 具有轻量级的特点，移动互联网的飞速发展轻量级的协议非常受欢迎，JSON 得到了广泛的应用
对资源的操作，Web 服务在该资源上所支持的一系列请求方法，比如：POST,GET,PUT,DELETE
```

#### 3、调用及调试API的几种方式

```
第一种方式：curl 命令（linux 下发送 HTTP 请求并接受响应的命令行工具），这种方式其实用的比较少，比较麻烦
第二种方式：比较常用的 OpenStack 命令行客户端，每一个 OpenStack 项目都有一个 Python 写的命令行客户端
第三种方式：用 Firefox 或 Chrome 浏览器的 REST 的客户端（图形界面的，浏览器插件）
第四种方式：用 OpenStack 的 SDK，可以不用手动写代码发送 HTTP 请求调用 REST 接口，还省去了一些管理诸如 Token 等数据的工作，能够很方便地基于 OpenStack 做开发，  那么 OpenStack 官方提供的是 Python 的 SDK，当然还有第三方提供的 SDK 比如说支持 Java 的著名的 Jclouds，还有支持 Node.js、Ruby、.Net 等等
```

### 三、openstack组件及内部通信

#### 1、cinder

**本身只是一个资源管理系统，通过插件的方式，结合不同后端存储的驱动提供块存储服务。**

主要组成：

- Cinder-api ：负责向外提供Cinder REST API
- Cinder-scheduler：负责分配存储资源
- Cinder-volume ：负责封装driver，不同的driver负责控制不同的后端存储

Cinder提供qos支持框架，具体的实现依赖于各vendor实现的plugin。

#### 2、neutron