---
layout: post
category: openstack
title: Placement服务解析
tagline: by 噜噜噜
tags: 
  - openstack
published: true
---



<!--more-->

https://blog.csdn.net/Jmilk/article/details/82980378

### 一、背景

随着 Resource Provider 的建模越来越复杂（起初只是将 Compute Node、Storage Pool 作为 Resource Provider，现在还将 NUMA、SR-IOV、Bandwidth 都建模成了 Resource Provider，而且还引入了 Nests Resource Provider 的实现），相应的 Allocation Candidates Filtering 的方式或策略也越来越丰富

OpenStack 除了要处理计算节点 CPU，内存，PCI 设备、本地磁盘等内部资源外，还经常需要纳管有如 SDS、NFS 提供的存储服务，SDN 提供的网络服务等外部资源

但在以往，Nova 只能处理由计算节点提供的资源。Nova Resource Tracker 假定所有资源均来自计算节点，因此在周期性上报资源状况时，Resource Tracker 只会单纯对计算节点清单进行资源总量和使用量的加和统计。显然，这无法满足上述复杂的生产需求，也违背了 OpenStack 一向赖以自豪的开放性原则。

综上，Placement 的诞生并且最终与 Nova 解耦是非常合理的。从 IaaS 的观念上看，在对资源进行思考与建模时不再以计算节点为中心，也是非常正确的。当资源类型和提供者变得多样时，自然就需求一种高度抽象且简单统一的管理方法，让用户和代码能够便捷的使用、管理、监控整个 OpenStack 乃至于所有外部的系统资源，这就是 Placement（放置）

### 二、基本概念

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/2019101015485771.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9pcy1jbG91ZC5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)

**Resource Provider**：资源提供者，实际提供资源的实体，例如：Compute Node、Storage Pool、IP Pool 等。

**Resource Class**：资源种类，即资源的类型，Placement 为 Compute Node 缺省了下列几种类型，同时支持 Custom Resource Classes。

**Inventory**：资源清单，资源提供者所拥有的资源清单，例如：Compute Node 拥有的 vCPU、Disk、RAM 等 inventories。

**Provider Aggregate**：资源聚合，类似 HostAggregate 的概念，是一种聚合类型。

**Traits**：资源特征，不同资源提供者可能会具有不同的资源特征。Traits 作为资源提供者特征的描述，它不能够被消费，但在某些 Workflow 或者会非常有用。例如：标识可用的 Disk 具有 SSD 特征，有助于 Scheduler 灵活匹配 Launch Instance 的请求。

**Resource Allocations**：资源分配状况，包含了 Resource Class、Resource Provider 以及 Consumer 的映射关系。记录消费者使用了该类型资源的数量。

### 三、placement在nova中的应用

##### 1、创建虚拟机流程图

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20190118184546242.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0ptaWxr,size_16,color_FFFFFF,t_70)



2、

