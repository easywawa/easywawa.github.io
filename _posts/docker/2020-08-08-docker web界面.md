---
layout: post
category: docker
title: docker 可视化工具Portainer
tagline: by 噜噜噜
tags: 
  - xx
published: true
---



<!--more-->

# 单机模式

### 安装docker

##### 清空环境

```
rpm -qa |grep docker

#如果上述有返回则进行卸载：
yum remove docker docker-common docker-selinux docker-engine 

#该命令只会卸载Docker本身，而不会删除Docker存储的文件，例如镜像、容器、卷以及网络文件等。这些文件保存在/var/lib/docker 目录中，需要手动删除
```

##### 配置docker-ce的yum源

```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo  #将repo文件中的gpgcheck关闭
yum clean all && yum makecache
```

##### 查看docker版本<可选>

```
yum list docker-ce --showduplicates | sort -r
```

##### 安装docker并配置镜像加速

```
yum install docker-ce  ##安装最新版本的Docker
yum install docker-ce-<VERSION> 安装指定版本
例如：yum install docker-ce-18.06.3.ce-3.el7
docker --version   ##查看安装的docker版本

curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io  ##配置镜像加速

##保险起见，多配几个
{"registry-mirrors": ["http://f1361db2.m.daocloud.io","https://registry.docker-cn.com","http://hub-mirror.c.163.com","https://docker.mirrors.ustc.edu.cn"]}

systemctl daemon-reload
systemctl restart docker 
```

### 查找docker Portainer 镜像并下载

```
docker search portainer
```

![portainer-image](https://i.loli.net/2020/08/08/CdO1gBMWrPijTcm.png)

```
docker pull portainer/portainer
docker images  ##查看下载好的镜像
```

### 运行该容器

```
docker run -d -p 9000:9000 --restart=always -v /var/run/docker.sock:/var/run/docker.sock --name prtainer  portainer/portainer
```

浏览器输入docker宿主机[http://IP:9000](http://ip:9000/)

![首次登录](https://i.loli.net/2020/08/08/PjsLoHEeTXWz9ZA.png)



### 使用

如果是单节点，则此处选择local单机。如果还有其他的节点机器，那么选择remote



# 集群模式

更多的情况下，我们会有一个`docker`集群，可能有几台机器，也可能有几十台机器，因此，进行集群管理就十分重要了，`Portainer`也支持集群管理，`Portainer`可以和`Swarm`一起来进行集群管理操作。这里我首先搭建了一个`Swarm`

`Swarm`集群的搭建方法可参考这篇文章：

[Swarm集群搭建](https://easywawa.github.io/2020/08/08/swarm%E9%9B%86%E7%BE%A4%E6%90%AD%E5%BB%BA/)

启动`Portainer`之后，首页还是给`admin`用户设置密码（这里和单机启动一样）。接下来是设置节点了，如下图：

**Endpoint URL** 是swarm集群中设置的节点URL，此处我的集群端口为2376

![初始界面](https://i.loli.net/2020/08/08/7ieFGusxkW3ItVr.png)



##### 使用方式

后续会更新

