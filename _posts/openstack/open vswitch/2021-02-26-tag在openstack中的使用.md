---
layout: post
category: open vswitch
title: tag在openstack中的使用
tagline: by 噜噜噜
tags: 
  - OVS
published: true
---



<!--more-->

openstack有多种网络插件，其中最重要的就是ovs，即openvswitch-plugin。在使用ovs实现openstack中的各种网络时，这里各种网络指：local，flat，vlan，vxlan等，tag标签的使用可以说是每一种网络都离不开的

#### 1、local网络

local网络是虚拟机的网络和网桥连接，但是网络和服务器网卡之间没有连接。流量限制在网桥内部。在local网络中，为了实现网络隔离，不同网络之间连接到网桥的tag是不一样的。在同一个tag下的网络可以互相通信，当然网络是访问不到外网的，则是local网络的最大特征。

#### 2、flat网络

flat网络叫平面网络即为不带tag的网络。不带也是一种特征。flat网络模式下，每创建一个网络，就需要独占一块网卡，所以一般也不会使用这种网络作为租户网络。虽然说flat网络不带tag，但是**其实是所有的port都使用了默认的tag号1**，所以能够看到网桥中port都有tag为1。

#### 3、vlan网络

在vlan网络中。每一个网络在br-int上的tag号都是不一样的，比如**使用网络1创建的虚拟机**，其port的tag是1，使用网络2创建的虚拟机，其port的tag是2。有了不同的tag就能够实现了vlan隔离

在br-int上定义的tag号不会考虑物理交换机上的vlan id支持。通俗来说就是ovs是虚拟交换机，tag号自己管理，而物理交换机的vlan id是物理交换机管理。这两个vlan是不同设备的，所有不能保证可以直接通用

**所以转换关系为：**

​	在br-ethx上对来自br-int 的数据，将vlan 1转化成物理网卡能通过的vlan 100

​	在br-int上对来自br-ethx的数据，将vlan 100转成ovs交换机能通过的vlan 1



如下的流表说明：

br-int上流表：

```
#ovs-ofctl dump-flows br-int
 cookie=0x0, duration=100.795s, table=0, n_packets=6, n_bytes=468, idle_age=90, priority=2,in_port=3 actions=drop
 cookie=0x0, duration=97.069s, table=0, n_packets=22, n_bytes=6622, idle_age=31, priority=3,in_port=3,dl_vlan=101 actions=mod_vlan_vid:1,NORMAL
 cookie=0x0, duration=95.781s, table=0, n_packets=8, n_bytes=1165, idle_age=11, priority=3,in_port=3,dl_vlan=102 actions=mod_vlan_vid:2,NORMAL
 cookie=0x0, duration=103.626s, table=0, n_packets=47, n_bytes=13400, idle_age=11, priority=1 actions=NORMAL
```



 br-ethx上流表：

```
#ovs-ofctl dump-flows br-eth0
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=73.461s, table=0, n_packets=51, n_bytes=32403, idle_age=2, hard_age=65534, priority=4,in_port=4,dl_vlan=1 actions=mod_vlan_vid:101,NORMAL
 cookie=0x0, duration=83.461s, table=0, n_packets=51, n_bytes=32403, idle_age=2, hard_age=65534, priority=4,in_port=4,dl_vlan=2 actions=mod_vlan_vid:102,NORMAL
 cookie=0x0, duration=651.538s, table=0, n_packets=72, n_bytes=3908, idle_age=2574, hard_age=65534, priority=2,in_port=4 actions=drop
 cookie=0x0, duration=654.002s, table=0, n_packets=31733, n_bytes=6505880, idle_age=2, hard_age=65534, priority=1 actions=NORMAL 
```

`mod_vlan_id:转换vlanID`

#### 4、vxlan网络

![img](https://img2020.cnblogs.com/blog/1060878/202005/1060878-20200501150228940-338857218.png)

在正常的网络封装上还有外层，并且重要的是中间还有一个vxlan头。重点就在这个vxlan的头，vxlan头部中有一个tunnel id。**不同的vxlan网络之间使用tunnel id来隔离**

创建虚拟机之后，在br-int上的port会有tag号。不同的网络之间tag号是不一样的。那么分情况讨论：

- 如果同一网络的虚拟机都在一个计算节点，同一个br-int上，它们之间的tag是一样的，所以直接通过br-int转发数据。不同网络之间tag不同，br-int根据tag实现隔离。
- 如果同一网络的虚拟机分布在不同的计算节点上，这时就需要通过bt-tun这个网桥发送出去。在br-tun上维护了一个vlan和vxlan之间的转换关系。同一个租户网络中在不同节点上对应的tag号不一样并不会影响正常的通信，因为在出服务器时br-tun已经将tag剥离，到了相应的服务器时会加上该tunnel id在该服务器上的对应的tag号

**看下将vlan与vxlan之间转换的流表：**

​	将vlan转化成vxlan

```
cookie=0x9814613d8b13e33b, duration=355743.467s, table=22, n_packets=121, n_bytes=5490, idle_age=65534, hard_age=65534, priority=1,dl_vlan=786 actions=strip_vlan,load:0x25a->NXM_NX_TUN_ID[],output:3,output:2,output:5,output:4
 cookie=0x9814613d8b13e33b, duration=335047.168s, table=22, n_packets=114, n_bytes=5460, idle_age=23232, hard_age=65534, priority=1,dl_vlan=788 actions=strip_vlan,load:0x222->NXM_NX_TUN_ID[],output:3,output:2,output:5,output:4
```

​	将vxlan 转化成vlan

```
cookie=0x9814613d8b13e33b, duration=355644.212s, table=4, n_packets=1025, n_bytes=107512, idle_age=17091, hard_age=65534, priority=1,tun_id=0x25a actions=mod_vlan_vid:786,resubmit(,10)
 cookie=0x9814613d8b13e33b, duration=334947.915s, table=4, n_packets=8487, n_bytes=710987, idle_age=38, hard_age=65534, priority=1,tun_id=0x222 actions=mod_vlan_vid:788,resubmit(,10)
```

