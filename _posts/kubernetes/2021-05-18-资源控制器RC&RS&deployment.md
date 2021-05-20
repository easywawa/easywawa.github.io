---
layout: post
category: kubernetes
title: 资源控制器RC、RS、deployment
tagline: by 噜噜噜
tags: 
  - RS
published: true
---



<!--more-->

### 一、名称概述

RC（ReplicationController）主要的作用就是用来确保容器应用的副本数始终保持在用户定义的副本数。即如果有容器异常退出，会自动创建新的Pod来替代

RS（Replicaset）也是一样的作用。官方建议使用RS来代替RC进行部署，**因为RS支持集合式的selector**

deployment是通过RS去创建和管理对应的pod以及不同的RS交替去完成滚动更新。

Deployment为Pod 和Replicaset 提供了一个声明式定义（declarative)方法，用来替代以前的

ReplicationController 来方便的管理应用。典型的应用场景包括：
　　·定义Deployment来创建Pod和ReplicaSet
　　·滚动升级和回滚应用
　　·扩容和缩容
　　·暂停和继续Deployment


