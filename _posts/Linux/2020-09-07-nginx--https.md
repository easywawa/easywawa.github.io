---
layout: post
category: Linux
title: nginx-https
tagline: by 噜噜噜
tags: 
  - nginx
published: true
---

<!--more-->

1、**安装依赖**

```
yum install -y gcc-c++ pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

2、**下载nginx**

```
wget http://nginx.org/download/nginx-1.10.2.tar.gz
tar -zxvf nginx-1.10.2.tar.gz
cd nginx-1.10.2/
```

3、**配置nginx**

配置HTTPS

```
./configure --prefix=/usr/local/nginx --with-http_ssl_module
```

4、**编译安装nginx**

```
make
make install
```

5、**设置nginx开机并启动，添加到系统服务**

```
vi /etc/init.d/nginx
#!/bin/sh  
# chkconfig: 2345 85 15  
# Startup script for the nginx Web Server  
# description: nginx is a World Wide Web server.  
# It is used to serve HTML files and CGI.  
# processname: nginx  
# pidfile: /usr/local/nginx/logs/nginx.pid  
# config: /usr/local/nginx/conf/nginx.conf  
  
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin  
DESC="nginx deamon"  
NAME=nginx  
DAEMON=/usr/local/nginx/sbin/$NAME  
SCRIPTNAME=/etc/init.d/$NAME  
  
test -x $DAEMON || exit 0  
  
d_start(){  
 $DAEMON || echo -n "already running"  
}  
  
d_stop(){  
 $DAEMON -s quit || echo -n "not running"  
}  
  
  
d_reload(){  
 $DAEMON -s reload || echo -n "can not reload"  
}  
  
case "$1" in  
start)  
 echo -n "Starting $DESC: $NAME"  
 d_start  
 echo "."  
;;  
stop)  
 echo -n "Stopping $DESC: $NAME"  
 d_stop  
 echo "."  
;;  
reload)  
 echo -n "Reloading $DESC conf..."  
 d_reload  
 echo "reload ."  
;;  
restart)  
 echo -n "Restarting $DESC: $NAME"  
 d_stop  
 sleep 2  
 d_start  
 echo "."  
;;  
*)  
 echo "Usage: $ScRIPTNAME {start|stop|reload|restart}" >&2  
 exit 3  
;;  
esac  
  
exit 0  



# 给启动文件添加执行权限
# chmod +x /etc/init.d/nginx  
 
# 添加开机自动启动nginx服务
# chkconfig --add nginx 
 
# 修改服务启动设置
# chkconfig nginx on 
 
# 显示开机可以自动启动的服务
# chkconfig --list nginx  
nginx 0:off 1:off 2:on 3:on 4:on 5:on 6:off
```

6、创建ssl证书

创建证书存放目录

```
mkdir /usr/local/nginx/conf/ssl
cd /usr/local/nginx/conf/ssl
```

创建服务器私钥：

```
openssl genrsa -des3 -out server.key 1024
```

【输入口令】

创建签名请求的证书（CSR）：

```
openssl req -new -key server.key -out server.csr  【输入上面设置的口令，其他全部回车】
openssl rsa -in server.key -out server_nopassword.key   #对key进行解密
openssl x509 -req -days 365 -in server.csr -signkey server_nopassword.key -out server.crt
```



![img](https://www.rishiqing.com/task/v1/folder/file/url?signature=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ0eXBlIjoidmlldyIsImlhdCI6MTU3NTI3OTQ3MSwiZmlsZUlkIjozMjM1Njg3LCJ0aW1lc3RhbXAiOjE1NzUyNzk0NzEzMDF9.xkN75_yZEwA8oxrpR9PU99E17L2jZI4BynegcjV_LGY)  



7.**编辑配置文件**

```
cd /usr/local/nginx/conf
vim nginx.conf
  server {
    listen    443 ssl;
    server_name localhost;

    ssl on;
    ssl_certificate   /usr/local/nginx/ssl/server.crt;
    ssl_certificate_key /usr/local/nginx/ssl/server_nopassword.key;

  #  ssl_session_cache  shared:SSL:1m;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;
  #  ssl_ciphers HIGH:!aNULL:!MD5;
  #  ssl_prefer_server_ciphers on;

    location / {
      root  html;
      index index.html index.htm;
    }
  }
```

7、相关操作命令

```
# service nginx start
# service nginx stop
# service nginx reload
```

8、添加html文件

```
cd /usr/local/nginx/html
vim Public_Base_Service.html
Crontroller Base Service
```

9、web访问