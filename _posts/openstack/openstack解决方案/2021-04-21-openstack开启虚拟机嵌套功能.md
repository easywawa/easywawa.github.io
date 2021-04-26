---
layout: post
category: openstack
title: openstack 虚拟机嵌套
tagline: by 噜噜噜
tags: 
  - openstack解决方案
published: true
---



<!--more-->

### 背景描述：

因业务需要，openstack环境需要实现虚拟机嵌套功能。

### 一、Linux Kernel开启嵌套

嵌套式虚拟nested是一个可通过内核参数来启用的功能。它能够使一台虚拟机具有物理机CPU特性,支持vmx或者svm(AMD)硬件虚拟化。关于nested的具体介绍,可以看这里 。该特性需要内核升级到Linux 3.X版本 ，所以在centos6下是需要先升级内核的，而在centos7下已默认支持该特性，不过默认是不开启的，需要通过修改参数支持。
启用Nested

```
echo 'options kvm_intel nested=1' >/etc/modprobe.d/kvm-nested.conf
```

卸载模块

```
modprobe -r kvm_intel
```

注：执行此步时需要当前节点上无虚拟机，有的话迁走。并且关闭libvirt服务

重新加载模块

```
modprobe kvm_intel
```

查看Nested是否启用成功

```
cat /sys/module/kvm_intel/parameters/nested
Y
```



### 二、nova 配置为host-passthrough

在nova的配置文件修改cpu mode

```
[libvirt] 

cpu_mode=host-passthrough
```

### 三、重启nova/libvirt服务

进入虚拟机查看

```
cat /proc/cpuinfo |grep vmx
```

![image-20210421223231074](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20210421223231074.png)