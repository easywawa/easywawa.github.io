---
layout: post
category: kubernetes
title: kubernetes对象介绍
tagline: by 噜噜噜
tags: 
  - xx
published: true
---



<!--more-->

### 一、kubernetes对象和对象模型

Kubernetes对象是一种持久化的、用于表示集群状态的实体。

一种声明式的意图的记录，一般使用yaml文件描述对象

Kubernetes集群使用Kubernetes对象来表示集群的状态

通过API/kubectl管理Kubernetes对象

![img](https://img2020.cnblogs.com/blog/1260387/202003/1260387-20200322170157959-1435288933.png)



### 二、常用的metadata属性

**Name和UID：**
　　Kubernetes集群中所有对象都通过name和UID明确标识。
　　在Kubernetes集群的整个生命周期内创建的每个对象实例都具有不同的UID

**Namespace：**
　　不仅仅是一个属性，本身也是一个object。
　　用于将物理集群划分为多个虚拟集群。
　　常用来隔离不同的用户及权限。
　　内置三个Namespaces：default、kube-system和kube-public

​        Node和PersistentVolume不属于任何namespace

**Label：**
　　用于建立集群对象之间的灵活的、松耦合的多维关联关系。
　　一个label是一个键值对，其中key、value均由用户自己定义。
　　label可以附着在任何对象上，每个对象也可以有任意个标签。
　　标签可在对象定义时附加上，也可以通过命令动态管理标签。
　　Label可以将有组织目的的结构映射到集群对象上，从而形成一个与现实世界管理结构同步对应松耦合的、多维的对象管理结构。
　　通过label select查询和刷选建立对象间的关系。

**Annotations（注解）：**
　　可以将任意非标识性元数据附加到对象上。也是**以键值对形式呈现**。
　　工具和库可以检索到并使用这些Annotation元数据。
　　将数据作为Annotation附着在对象上，有利于创建一些用于部署、管理和做内部检查的共享工具或客户端

### 三、kubernetes对象分类

![img](https://img2020.cnblogs.com/blog/1260387/202003/1260387-20200322171125180-918325712.png)

#### 1、workload

**以pod的为中心的的一些对象：**

deployment/replicaset/replication controller :关于副相关的

daemonset：确保一些或全部node上运行某个pod的副本。

statefulset：有状态的集合，管理所有有状态的服务

cronjob/job：定时任务\一次性任务



#### 2、discovery&load balance

**service（SVC）**：四种类型 

[实践]: http://www.yoyoask.com/?p=2192

- ​	**ClusterIp**: 默认类型，这种service有一个vip来进行访问。有种特殊的clusterip类型--> Headless Service，是没有IP的。

- ​	**NodePort**：nodePort 的原理在于在 node 上开了一个端口，将向该端口的流量导入到 kube-proxy，然后  由 kube-proxy 进一步到给对应的 pod。（能够将内部服务暴露给外部的一种方式)

  ​	 固定范围在kube-apiserver配置文件下参数。也可以在yaml中指定固定端口。

  ```
     --service-node-port-range=30000-50000
  ```

  

- ​	**LoadBalance**：loadBalancer 和 nodePort 其实是同一种方式。都是向外部暴露一个端口，以提供访问，区别在于 loadBalancer 比 nodePort 多了一步，就是可以调用cloud provider 去创建 LB 来向节点导流，通俗说就是前面端口访问加了负载均衡LoadBalancer(收费服务，负载均衡需要收费，按流量收费的，很贵)

- ​	**ExternalName**：这种类型的 Service 通过返回 CNAME 和它的值，可以将服务映射到 externalName 字段的内容



**Endpoint:**

​	endpoint是k8s集群中的一个资源对象，存储在etcd中，用来记录一个service对应的所有pod的访问地址。**service配置selector，endpoint controller才会自动创建对应的endpoint对象**

例如，k8s集群中创建一个名为hello的service，就会生成一个同名的endpoint对象，ENDPOINTS就是service关联的pod的ip地址和端口

**ingress:** 

​	在集群服务很多的时候，相对于NotePort, LoadBalance占用很多集群机器的端口和每个service一个LB的浪费，ingress则只需要一个NodePort或者一个LB就可以满足所有service对外服务的需求

#### 3、config & storage

**configmap**: 

​	常用来向Pod提供非敏感的配置信息。

​	ConfigMap可以通过三种方式在Pod中使用：环境变量、命令行参数或以文件形式通过数据卷插件挂载到Pod中

**secret**: 

​	解决的是集群内密码、token、密钥等敏感的数据的配置问题。

​	

**volume**: 

**persistentvolumeclaim**: 

#### 4、cluster

​	**Node/Namespace**

​    **PersistentVolume**

​	**ClusterRole/ClusterRoleBinding**

​	**ResourceQuota**