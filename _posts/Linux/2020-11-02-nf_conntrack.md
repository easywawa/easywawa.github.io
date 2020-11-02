---
layout: post
category: Linux
title: nf_conntrack 的解读
tagline: by 噜噜噜
tags: 
  - conntrack
published: true
---



<!--more-->

问题现象：在 `dmesg` 或 `/var/log/messages` 看到大量以下记录：

kernel: nf_conntrack: table full, dropping packet

**原因：服务器访问量大，内核 netfilter 模块 conntrack 相关参数配置不合理，导致 IP 包被丢掉，连接无法建立**

### 一、概念解读

很多人以为 Linux 经过这么多年优化，默认参数应该“足够好”，其实不是。默认参数面向“通用”服务器，不适用于连接数和访问量比较多的场景

#### 1、作用

nf_conntrack 模块用于跟踪连接的状态，供其他模块使用。

需要 NAT 的服务都会用到它，例如防火墙、Docker 等。以 iptables 的 `nat` 和 `state` 模块为例：

- `nat`：根据转发规则修改 IP 包的源/目标地址，靠 conntrack 记录才能让返回的包能路由到发请求的机器。
- `state`：直接用 conntrack 记录的连接状态（`NEW`/`ESTABLISHED`/`RELATED`/`INVALID` 等）来匹配防火墙过滤规则

#### 2、工作方式

nf_conntrack 跟踪**所有**网络连接，记录存储在 1 个哈希表里

即使来自客户端的访问量不多，内部请求多的话照样会塞满哈希表，例如 ping 本机也会留下这么一条记录：

```
ipv4 2 icmp 1 29 src=127.0.0.1 dst=127.0.0.1 type=8 code=0 id=26067 src=127.0.0.1 dst=127.0.0.1 type=0 code=0 id=26067 mark=0 use=1
```

#### 3、清除记录

连接记录会在哈希表里保留一段时间，**根据协议和状态有所不同**，直到超时都没有收发包就会清除记录

如果服务器比较繁忙，新连接进来的速度远高于释放的速度，把哈希表塞满了，新连接的数据包就会被丢掉。这样就会造成问题

### 二、查看参数

#### 1、查看是否开启了netfilter.conntrack

```
sysctl -a | grep conntrack  ##如果没有任何输出，则表示未启用
```

#### 2、查看超时相关的参数

所谓超时是清除 conntrack 记录的秒数，从某个连接收到最后一个包后开始倒计时， 倒数到 0 就会清除记录，**中间收到包会重置**。不同协议的不同状态有不同的超时时间

```
sysctl -a | grep conntrack | grep timeout
```

![](https://i.loli.net/2020/11/02/bR14CQHMulxg7wy.png)

#### 3、查看哈希表设置

查看哈希表大小（桶的数量）：**【HASHSIZE】**

```
sysctl net.netfilter.nf_conntrack_buckets  ##只读
```

查看最大跟踪连接数：**【CONNTRACK_MAX】**

```
sysctl net.netfilter.nf_conntrack_max
```

**\# 默认 nf_conntrack_max = nf_conntrack_buckets * 4**

\# max 是 bucket 的多少倍决定了每个桶里的链表有多长，**因此默认链表长度为 4**。# 比较早的版本是 8



查看netfilter模块加载时候的默认值：

```
dmesg | grep conntrack
[95018.129229] nf_conntrack version 0.5.0 (65536 buckets, 262144 max)
```



#### 4、查看哈希表使用情况

```
sysctl net.netfilter.nf_conntrack_count  ##只读
```

**cat  /proc/net/nf_conntrack  都在这里面记录的**

![](https://i.loli.net/2020/11/02/6YMnmHAyh4jZreC.png)



这个值跟 `net.netfilter.nf_conntrack_buckets` 的值比较，当`nf_conntrack_count` 的值持续超过 `nf_conntrack_max` 的 20% 就该考虑扩容

因为 bucket 的值默认是 max 的 25%，用了 max 的 20% 也就是 80% 的桶都有元素了

### 三、内存与哈希表的关系

#### 1、CONNTRACK_MAX的默认值: （最大跟踪连接数）

```
在32位架构上，CONNTRACK_MAX = RAMSIZE (以bytes记) / 16384 = RAMSIZE (以MegaBytes记) * 64

在64位架构上，CONNTRACK_MAX = RAMSIZE (in bytes) / 16384 / 2 = RAMSIZE (以MegaBytes记) * 32
```

注：对于带有超过1G内存的系统，CONNTRACK_MAX的默认值会被限制在65536

#### 2、HASHSIZE的默认值：（桶数量）

通常，CONNTRACK_MAX = HASHSIZE * 4，每个链接的列表平均包含4个conntrack的条目，即每个桶4个条目。以前是8个

```
在32位架构上，HASHSIZE = CONNTRACK_MAX / 4 = RAMSIZE (以bytes记) / 65536 = RAMSIZE (以MegaBytes记) * 16

在64位架构上，HASHSIZE = CONNTRACK_MAX / 4 = RAMSIZE (以bytes记) / 65536 / 2 = RAMSIZE (以MegaBytes记) * 8
```

#### 3、每个桶中的条目数设置

一个链接列表（CONNTRACK_MAX/HASHSIZE的值在优化的状态下并且达到上限）长度的平均值不能设置太大。这个比值默认值是4，以前的版本是8

**在系统有足够的内存并且性能真的很重要的时候，你可以试着使平均值是一个跟踪连接条目配一个哈西桶，这意味着HASHSIZE = CONNTRACK_MAX**

### 四、调整内核参数

#### 1、调整的方式：

​	哈希表扩容（nf_conntrack_buckets、nf_conntrack_max）

​	设置超时相关的参数

#### 2、影响

主要是内存使用增加。32 位系统还要关心内核态的地址空间够不够

netfilter 的哈希表存储在内核态的内存空间，这部分内存不能 swap，操作系统为了兼容 32 位，默认值往往比较保守。

32 位系统的虚拟地址空间最多 4G，其中内核态最多 1G，通常能用的只有前 896M。
给 netfilter 分配太多地址空间可能会导致其他内核进程不够分配。1 条跟踪记录 300 字节左右，因此当年 nf_conntrack_max 默认 65535 条，占 20多MB。
**64 位系统的虚拟地址空间有 256TB，内核态能用一半，只需要关心物理内存的使用情况。**



### 五、计算方式

**1、根据内存计算CONNTRACK_MAX和HASHSIZE的值（这里以64位为例）**

CONNTRACK_MAX = RAMSIZE (以MegaBytes记) * 32

HASHSIZE = RAMSIZE (以MegaBytes记) * 8

#### 2、占用内存总量

根据计算公式：

```
size_of_mem_used_by_conntrack (in bytes) = CONNTRACK_MAX * sizeof(struct ip_conntrack) + HASHSIZE * sizeof(struct list_head)
```

- ```
  sizeof(struct ip_conntrack)
  ```

   在不同架构、内核版本、编译选项下不一样。这里按 352 字节算。

  - 老文章说模块启动时会在 syslog 里打印这个值，但现在没有。

- `sizeof(struct list_head) = 2 * size_of_a_pointer`（32 位系统的指针大小是 4 字节，64 位是 8 字节）

则一台64位系统，756G内存的物理机，占用内存大小为:

CONNTRACK_MAX = 25208064

HASHSIZE = 6302016

size_of_mem_used_by_conntrack  = 25208064 * 352 +  6302016 * 8  ≈ 8G







