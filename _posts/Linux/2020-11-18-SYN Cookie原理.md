---
layout: post
category: Linux
title: SYN Cookie的原理
tagline: by 噜噜噜
tags: 
  - xx
published: true
---



<!--more-->

SYN Flood是一种非常危险而常见的Dos攻击方式。到目前为止，能够有效防范SYN Flood攻击的手段并不多，

SYN Cookie就是其中最著名的一种

### 一、SYN Flood

##### 1、SYN Flood的概念

SYN Flood攻击是一种典型的**拒绝服务**(Denial of Service)攻击。所谓的拒绝服务攻击就是通过进行攻击，使受害主机或网络不能提供良好的服务，从而间接达到攻击的目的

##### 2、SYN Flood攻击的原理

SYN Flood攻击利用的是IPv4中TCP协议的三次握手(Three-Way Handshake)过程进行的攻击

在最常见的SYN Flood攻击中，攻击者在短时间内发送大量的TCP SYN包给受害者，这时攻击者是TCP客户机，受害者是TCP服务器。根据上面的描述，**受害者会为每个TCP SYN包分配一个特定的数据区，只要这些SYN包具有不同的源地址（这一点对于攻击者来说是很容易伪造的）。这将给TCP服务器系统造成很大的系统负担，最终导致系统不能正常工作**

【从[backlog的意义](https://segmentfault.com/a/1190000019252960)中可知，服务器能保存的半连接的数量是有限的！所以当服务器受到大量攻击报文时，它就不能再接收正常的连接了。换句话说，它的服务不再可用了！这就是`SYN Flood`攻击的原理，它是一种典型的`DDoS`攻击】

**在Linux内核2.2之前，backlog大小包括半连接状态和全连接状态两种队列大小**

**注：**

**三次握手**：大家知道协议规定，如果一端想向另一端发起TCP连接，它需要首先发送TCP SYN 包到对方，对方收到后发送一个TCP SYN+ACK包回来，发起方再发送TCP ACK包回去，这样三次握手就结束了。我们把TCP连接的发起方叫作"TCP客户机（TCP Client）"，TCP连接的接收方叫作"TCP服务器（TCP Server）"。

在TCP服务器收到TCP SYN request包时，在发送TCP SYN+ACK包回TCP客户机前，**TCP服务器要先分配好一个数据区专门服务于这个即将形成的TCP连接**。一般把收到SYN包而还未收到ACK包时的连接状态成为半开连接（Half-open Connection）。在这个过程中，如果服务器一直没有收到`ACK`报文(比如在链路中丢失了)，服务器会在超时后重传`SYN+ACK`。如果经过多次超时重传后，还没有收到, 那么服务器会回收资源并关闭`半连接`，仿佛之前最初的`SYN`报文从来没到过一样

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20190526105051890.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NoZW5tbzE4N0ozWDE=,size_16,color_FFFFFF,t_70)

### 二、SYN Cookie

##### 1、SYN Cookie原理

它的原理是，在TCP服务器收到TCP SYN包并返回TCP SYN+ACK包时，不分配一个专门的数据区，而是根据这个SYN包计算出一个cookie值。在收到TCP ACK包时，TCP服务器在根据那个cookie值检查这个TCP ACK包的合法性。如果合法，再分配专门的数据区进行处理未来的TCP连接

##### 2、Cookie的计算

服务器收到一个SYN包，计算一个消息摘要mac。

mac = MAC(A, k);

MAC是密码学中的一个消息认证码函数，也就是满足某种安全性质的带密钥的hash函数，它能够提供cookie计算中需要的安全性。

在Linux实现中，MAC函数为SHA1。

A = SOURCE_IP || SOURCE_PORT || DST_IP || DST_PORT || t || MSSIND

k为服务器独有的密钥，实际上是一组随机数。

t为系统启动时间，每60秒加1。

MSSIND为MSS对应的索引。

##### 3、缺点

①由于cookie的计算只涉及到包头部分信息，在建立连接的过程中不在服务器端保存任何信息，所以失去了协议的许多功能，比如超时重传。

②由于计算cookie有一定的运算量，增加了连接建立的延迟时间，因此，SYN Cookie技术不能作为高性能服务器的防御手段



参考：https://blog.csdn.net/chenmo187j3x1/article/details/90573816