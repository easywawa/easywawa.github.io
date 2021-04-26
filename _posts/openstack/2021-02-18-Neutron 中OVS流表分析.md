---
layout: post
category: openstack
title: Neutron VXLAN/GRE模式中br-tun流表分析
tagline: by 噜噜噜
tags: 
  - xx
published: true
---



<!--more-->

### 一、neutron中的bridge

#### 1、bridge类别

neutron中主要有br-int、br-tun和br-phy几个bridge。

br-int是所有vm连的bridge，br-tun在使用tunnel的时候会用到，br-phy在使用vlan模式的时候会用到。

#### 2、bridge模式

OpenvSwitch中 bridge 有两种模式：“normal” 和 “flow”，“normal” 模式的 bridge 同普通的L2交换机数据转发模式一样，“flow” 模式的 bridge 是**根据其流表（flow tables） 来进行转发的**

- br-int 是一个 “normal” 模式的虚拟网桥
- br-tun 是 “flow” 模式的



### 二、neutron设计的网络技术

- bridge：网桥，Linux中用于表示一个能连接不同网络设备的虚拟设备，linux中传统实现的网桥类似一个hub设备，而ovs管理的网桥一般类似交换机
- br-int：综合网桥，常用于表示实现主要内部网络功能的网桥
- br-ex：外部网桥，通常表示负责跟外部网络通信的网桥
- GRE：一种通过封装来实现隧道的方式。在openstack中一般是基于L3的gre，即original pkt/GRE/IP/Ethernet
- VETH：虚拟ethernet接口，通常以pair的方式出现，一端发出的网包，会被另一端接收，可以形成两个网桥之间的通道
- qvb：neutron veth, Linux Bridge-side
- qvo：neutron veth, OVS-side
- TAP设备：模拟一个二层的网络设备，可以接受和发送二层网包
- TUN设备：模拟一个三层的网络设备，可以接受和发送三层网包
- Vlan：虚拟 Lan，同一个物理 Lan 下用标签实现隔离，可用标号为1-4094
- VXLAN：一套利用 UDP 协议作为底层传输协议的 Overlay 实现。一般认为作为 VLan 技术的延伸或替代者
- namespace：用来实现隔离的一套机制，不同 namespace 中的资源之间彼此不可见