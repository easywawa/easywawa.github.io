---
layout: post
category: kubernetes
title: Docker 和 kubelet 的 cgroup driver
tagline: by 噜噜噜
tags: 
  - k8s
published: true
---



<!--more-->

### 一、两种驱动的介绍

安装k8s的时候，有一个强烈的要求就是docker与kubelet的驱动，必须是保持一致的，需要都使用system或cgroupfs

那么 systemd 和 cgroupfs 这两种驱动有什么区别呢？

1. systemd cgroup driver 是 systemd 本身提供了一个 cgroup 的管理方式，使用systemd 做 cgroup 驱动的话，所有的 cgroup 操作都必须通过 systemd 的接口来完成，不能手动更改 cgroup 的文件

2. cgroupfs 驱动就比较直接，比如说要限制内存是多少、要用 CPU share 为多少？直接把 pid 写入对应的一个 cgroup 文件，然后把对应需要限制的资源也写入相应的 memory cgroup 文件和 CPU 的 cgroup 文件就可以了。所以可以看出来 systemd 更加安全，因为不能手动去更改 cgroup 文件，当然我们也推荐使用 systemd 驱动来管理 cgroup

![img](https://img2020.cnblogs.com/blog/1551569/202009/1551569-20200928160302987-625906156.png)



### 二、注意事项

根据官网的说法:

![image-20210421142614512](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210421142614512.png)

最好不要更改已经加入集群的节点的驱动，即使是重启kubelet也不能解决某些问题



![image-20210421143049489](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210421143049489.png)

### 三、配置cgroups

https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/