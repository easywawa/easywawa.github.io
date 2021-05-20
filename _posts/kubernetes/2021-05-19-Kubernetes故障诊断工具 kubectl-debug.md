---
layout: post
category: kubernetes
title: pod故障诊断工具：kubectl-debug
tagline: by 噜噜噜
tags: 
  - kubectl-debug
published: true
---



<!--more-->

介绍地址：

https://mp.weixin.qq.com/s?__biz=MzA5OTAyNzQ2OA%3D%3D&idx=1&mid=2649702098&scene=21&sn=daec7580ce03c8eccd81d24c93c682c7#wechat_redirect

#### 一、安装

curl -Lo kubectl-debug.tar.gz https://github.com/aylei/kubectl-debug/releases/download/v${PLUGIN_VERSION}/kubectl-debug_${PLUGIN_VERSION}_linux_amd64.tar.gz

如：

```
curl -Lo kubectl-debug.tar.gz https://github.com/aylei/kubectl-debug/releases/download/v0.1.1/kubectl-debug_0.1.1_linux_amd64.tar.gz
```

解压：

```
tar -zxvf kubectl-debug.tar.gz kubectl-debug
sudo cp kubectl-debug /usr/local/bin/
```

kubectl debug 默认使用 nicolaka/netshoot[6] 作为默认的基础镜像，里面内置了相当多的排障工具，下载镜像：

```
docker pull nicolaka/netshoot:latest
```



#### 二、测试

```
[root@mater1 ~]# kubectl debug coredns-7ff77c879f-vk49q -n kube-system --agentless --port-forward

Agent Pod info: [Name:debug-agent-pod-c75a194e-b885-11eb-9099-fa163e59f35d, Namespace:default, Image:aylei/debug-agent:latest, HostPort:10027, ContainerPort:10027]
Waiting for pod debug-agent-pod-c75a194e-b885-11eb-9099-fa163e59f35d to run...
pod coredns-7ff77c879f-vk49q PodIP 10.244.0.3, agentPodIP 192.168.1.25
wait for forward port to debug agent ready...
Forwarding from 127.0.0.1:10027 -> 10027
Forwarding from [::1]:10027 -> 10027
Handling connection for 10027
                             container created, open tty...
bash-5.1#

```

```
bash-5.1# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc noqueue state UP group default
    link/ether b6:b6:1a:ac:26:1f brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.244.0.3/24 brd 10.244.0.255 scope global eth0  
       valid_lft forever preferred_lft forever
```



常见的基础排障：

```
使用 iftop 查看容器网络流量：iftop -i  eth0

使用 drill 诊断 DNS 解析：drill -V 5 demo-service

使用 tcpdump 抓包：tcpdump -i eth0 -c 1 -Xvv
```

**诊断 CrashLoopBackoff**



排查 CrashLoopBackoff 是一个很麻烦的问题，Pod 可能会不断重启， kubectl exec 和 kubectl debug 都没法稳定进行排查问题，基本上只能寄希望于 Pod 的日志中打印出了有用的信息。为了让针对 CrashLoopBackoff 的排查更方便， kubectl-debug 参考 oc debug 命令，添加了一个 --fork 参数。当指定 --fork 时，插件会复制当前的 Pod Spec，做一些小修改， 再创建一个新 Pod：



- 新 Pod 的所有 Labels 会被删掉，避免 Service 将流量导到 fork 出的 Pod 上
- 新 Pod 的 ReadinessProbe 和 LivnessProbe 也会被移除，避免 kubelet 杀死 Pod
- 新 Pod 中目标容器（待排障的容器）的启动命令会被改写，避免新 Pod 继续 Crash



接下来，我们就可以在新 Pod 中尝试复现旧 Pod 中导致 Crash 的问题。为了保证操作的一致性，可以先 chroot 到目标容器的根文件系统中：



```
➜  ~ kubectl debug demo-pod --fork
root @ / [4] 🐳  → chroot /proc/1/root
root @ / [#] 🐳  → ls bin            entrypoint.sh  home           lib64          mnt            root           sbin           sys            tmp            var dev            etc            lib            media          proc           run            srv            usr
root @ / [#] 🐳  → ./entrypoint.sh# 观察执行启动脚本时的信息并根据信息进一步排障
```





#### 三、更多的工具

https://blog.csdn.net/M2l0ZgSsVc7r69eFdTj/article/details/103692433