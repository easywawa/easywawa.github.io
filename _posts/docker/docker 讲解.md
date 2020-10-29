---
layout: post
category: xx
title: docker 入门
tagline: by 噜噜噜
tags: 
  - xx
published: true
---



<!--more-->

### 一、允许远程客户端连接

默认配置下，Docker daemon 只能响应来自本地 Host 的客户端请求。如果要允许远程客户端请求，需要在配置文件中打开 TCP 监听，步骤如下：

```
1、编辑配置文件 /usr/lib/systemd/system/docker.service
  在环境变量 `ExecStart` 后面添加 `-H tcp://0.0.0.0`，允许来自任意 IP 的客户端连接
  ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0 -H fd:// --containerd=/run/containerd/containerd.sock
2、重新加载并重启
systemctl daemon-reload
systemctl restart docker
3、查看docker进程，发现docker守护进程在已经监听2375的tcp端口
/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H fd:// --containerd=/run/containerd/containerd.sock
4、验证（在浏览器上）
```

![](https://i.loli.net/2020/10/29/PtTBIWOF2pUi4MQ.png)

