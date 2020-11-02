---
layout: post
category: docker
title: dockerfile
tagline: by 噜噜噜
tags: 
  - docker
published: true
---



<!--more-->

### 一、常用指令

**FROM**
指定 base 镜像

**MAINTAINER**
设置镜像的作者，可以是任意字符串

**COPY**
将文件从 build context 复制到镜像

COPY 支持两种形式：

1. COPY src dest
2. COPY ["src", "dest"]

注意：src 只能指定 build context 中的文件或目录

**ADD**
与 COPY 类似，从 build context 复制文件到镜像。不同的是，如果 src 是归档文件（tar, zip, tgz, xz 等），文件会被自动解压到 dest

**ENV**
设置环境变量，环境变量可被后面的指令使用

**EXPOSE**
指定容器中的进程会监听某个端口，Docker 可以将该端口暴露出来

**VOLUME**
将文件或目录声明为 volume

**WORKDIR**
为后面的 RUN, CMD, ENTRYPOINT, ADD 或 COPY 指令设置**镜像中的当前工作目录**

**RUN**
在容器中运行指定的命令

**CMD**
容器启动时运行指定的命令。
Dockerfile 中可以有多个 CMD 指令，但只有最后一个生效。CMD 可以被 docker run 之后的参数替换

**ENTRYPOINT**
设置容器启动时运行的命令。
Dockerfile 中可以有多个 ENTRYPOINT 指令，但只有最后一个生效。CMD 或 docker run 之后的参数会被当做参数传递给 ENTRYPOINT



### 二、 CMD 和 RUN、ENTRYPOINT 的区别

RUN命令适用于在 docker build **构建docker镜像时执行的命令**

CMD命令是在 docker run 执行docker镜像**构建容器时使用**，可以动态的**覆盖CMD执行的命令**

CMD命令是用于默认执行的，且如果写了多条CMD命令，则只会执行最后一条

如果后续**存在ENTRYPOINT命令，则CMD命令或被充当参数或者覆盖**，而且Dockerfile中的**CMD命令最终可以被在执行 docker run命令时添加的命令所覆盖**。而ENTRYPOINT命令则是**一定会执行的**，一般用于执行脚本

```
#shell写法
FROM centos
CMD echo 'hello'

#exec写法
FROM centos
CMD ["echo","hello"]
```

1、在 shell 写法环境下

**如果存在 ENTRYPOINT命令**，则不管是在Dockerfile中存在CMD命令也好，还是在 docker run执行的后面添加的命令也好，**都不会被执行**

**如果不存在 ENTRYPOINT命令**，则可以被 docker run后面设置的命令覆盖，实现动态执行命令操作

2、在 exec 写法环境下

如果存在 ENTRYPOINT命令，则在Dockerfile中如果存在CMD命令或者是在 docker run执行的后面添加的命令，会被当做 ENTRYPOINT命令的参数来使用



