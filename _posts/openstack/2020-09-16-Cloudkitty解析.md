---
layout: post
category: openstack
title: Cloudkitty解析
tagline: by 噜噜噜
tags: 
  - xx
published: true
---

<!--more-->

主要参考：https://www.cnblogs.com/menkeyi/p/6972732.html

基于T版：https://docs.openstack.org/cloudkitty/train/admin/architecture.html

### 一、概述

Cloudkitty可以完成虚拟机实例(compute)、云硬盘(volume)、镜像(image)、网络进出流量(network.bw.in, network.bw.out)、浮动IP(network.floating)的计费

Cloudkitty主要依赖于遥测相关的项目，包括ceilometer和gnocchi，甚至是将要使用panko；计费策略和hashmap计费模型是其核心；模块插件化是其设计灵魂

### 二、架构分析

整体架构包括四部分：

-  对象获取（Tenant Fetcher），即获取计费租户
- 数据源的收集（Collector），即需要计费的数据
- 计费引擎（Rating），即计费标准
- 数据的存储（Storage）

![](https://w9.sanwen8.cn/mmbiz_png/Hl54p4MexNdoo9uXHRSLfNR4pwkG7teZuiaqeq8FI1QB19vIpdPI7TYUdJZph7F2ck4Vf0c8vQCe3yBTJsiadbag/640?wx_fmt=png)

##### 1、计费服务对象Tenant Fetcher

Cloudkitty要知道需要对谁的资源进行计费，有四种方式：

- Gnocchi

- 从keystone中获取（默认的方式）

  **具体逻辑是检查Cloudkitty用户是否在某个tenant内并拥有rating角色**

  ```
  keystone user-role-add --user cloudkitty --role rating --tenant demo
  ##将Cloudkitty用户加入租户并赋予rating角色
  ```

   cloudkitty.conf配置文件中 [fetcher_keystone]区域
  
  ```
  keystone_version：Defaults to `2`. Keystone version to use
  
  auth_section：keystone_authtoken
  ```
  
  
  
- Prometheus

- Source

更多配置：https://docs.openstack.org/cloudkitty/train/admin/configuration/fetcher.html



##### 2、数据收集Collector

这一步除了计费源数据的收集，还有一个十分重要的工作就是把这些数据转化为统一格式输入给计费引擎。

Cloudkitty中的transformer模块将针对不同collector的不同类型的服务数据进行格式统一，分为两个阶段：

1. 第一个阶段是将资源数据通过对应的CeilometerTransformer或者GnocchiTransformer进行转换
2. 第二阶段是使用CloudKittyFormatTransformer统一数据格式为data = [{'usage': {'service_type': [{'vol': {'unit': xx, 'qty': xxx}, 'desc': {'xxx': 'xxx',…, 'metadata': {‘xxx’: ’xxx’}}}]}, 'period': {'begin': xxxxx, 'end': xxxxx}}]交付给计费引擎。当然在不需要对资源数据进行转化的情况下，也可以直接使用CloudKittyFormatTransformer将数据转化为计费引擎所能识别的格式

**CloudKitty中提供了三个收集器：**

- gnocchi：Defaults to `gnocchi`
- monasca:
- prometheus:

更多配置：https://docs.openstack.org/cloudkitty/train/admin/configuration/collector.html



##### 3、计费引擎Rating

在计费策略上Cloudkitty采用的是周期轮询计费的方式，**默认是每一小时**对云环境中所需计费的资源进行费用核算，并将费用数据持久化存储起来

Cloudkitty当前的计费模型有三个，分别是noop，hashmap和pyscripts：

- noop模型为空，仅作为测试用
- hashmap是当前Cloudkitty实用价值最高，最容易使用，最接近实际案例的计费模型 
- pyscripts计费模型提供了使用python代码定制计费的接口，用户可以直接将含有计费逻辑的python脚本上传给cloudkitty实现定制化的计费



##### 4、费用数据的存储Storage

Cloudkitty的计费引擎是周期性计算资源费用的，而这个周期可以根据实际情况修改，**只要能在一个计费周期内完成所有资源的费用计算即可**

如果将计费周期调得更小，云环境中的需要计费的资源更多，那么将会对费用数据的存储带来挑战，它会直接影响到数据的使用效果，因此费用数据的存储也是一个重要的环节

Cloudkitty的storage目前支持sqlalchemy和gnocchi_hybrid两种方式存放费用数据，因为它们都是通过插件方式实现的，所以具体采用哪种可以**通过配置文件指定**  

```
 [storage]
backend = influxdb
version = 2
```

`version`: Defaults to 2. Version of the storage interface to use (must be 1 or 2).

**backend**:

* v1: sqlalchemy
* v2： influxdb、elasticsearch

更多配置：https://docs.openstack.org/cloudkitty/train/admin/configuration/storage.html



### 三、Hashmap计费模型

![](https://w9.sanwen8.cn/mmbiz_png/Hl54p4MexNdoo9uXHRSLfNR4pwkG7teZtmdB33oBTGnIpMIrfsX8o9WeCiaNr1HypiaN88uJJylShbmR8kuJibK2g/640?wx_fmt=png)

- Hashmap Group：一个Group表示一组计费规则，例如你可以创建多个计费规则，将它们分成两组来分别为instance和volume计费，避免计费规则混乱。

 

- Hashmap Service：一个Service将计费规则映射到具体的数据collector，例如compute，volume，image，network.bw.in，network.bw.out，network.floating。

 

- Hashmap Field：一个Field通常是指某个资源元数据metadata字典中的一个字段。例如对于instance，可以指定Field为flavor，availability_zone等；对于volume可以指定Field为volume_type，availability_zone和status等。既可以为Field指定mapping类型的计费规则也能指定Threshold类型的计费规则。

 

- Hashmap Mapping： 一个Mapping是指某个最终的计费规则，mapping这类计费规则可以为某个服务指定价格，也可以为Field指定价格。例如你可以指定compute服务的基础价格Flat=10$，这会适用于所有的云主机；同时你也能具体指定flavor name，value=m1.tiny，Rate=1.2，则flavor为m1.tiny的云主机价格就为10$*1.2=12$，value=m1.medium，Flat=20$，则flavor为m1.medium的云主机价格就是20$；再例如指定volume的基础单价为2$，选定Field为volume_type，SATA类型打九五折value=sata，Rate=0.95,则sata盘的单价就是2$*0.95=1.9$；SAS为标准云硬盘不打折，则不用设定，单价依然为2$；SSD为高速硬盘应加价0.2倍value=ssd，Rate=1.2，则ssd盘的单价就是2$*1.2=2.4$。

 

- Hashmap Threshold：Threshold是另外一种最终计费规则，threshold这类计费规则更适用于基于level的Rate价格。例如云硬盘基础单价为2$（可用mapping设定），如果用户购买超过50GB，可以打九折Level=50,Rate=0.9，此时单价为2$*0.9=1.8$；超过100GB打八折Level=100，Rate=0.8，此时单价为2$*0.8=1.6$。



官网用户使用：https://docs.openstack.org/cloudkitty/latest/user/rating/hashmap.html