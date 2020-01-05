---
title: 使用alpine构建nginx镜像
date: 2016-10-31 21:34:47
tags:
- docker
category:
- 服务器
- docker
---

[alpine linux](http://alpinelinux.org)作为一款Linux发行版，基于其安全,轻量和
完善的包管理的特性，常被用于于各种路由器和嵌入式设备中中，在docker容器的
编制中，也经常被用来作为基础镜像。本实例我将使用alpine构建一个基础的nginx镜
像,安装ssh服务用于容器的管理，并使用supervisord进行服务的注册和管理。

<!-- more -->

## 安装基本软件 ##
1. 修改软件源，并安装基础软件
    1.1  由于alpine官方的源较慢，我们选择[中科大](https://lug.ustc.edu.cn/wiki/mirrors/help/alpine) 的源作为软件源 
    1.2  由于我们选择编译的方式进行nginx的安装，安装软件部分选取gcc libc-dev 
    make,linux-headers 等工具
    1.3  nginx的安装上我们需要使用到ssl,正则，压缩等特性，我选择了openssl-dev, \
    pcre-dev,zlib-dev, gd-dev
    1.4  相对传统的内存管理，jemalloc提供了更高效的内存管理，这里选取jemalloc-dev
    软件包
    1.5  由于需要向nginx添加一些的扩展,需要使用到github提供的服务，选取git,curl等工具
    1.6  为了使用ssh， 需要安装openssh，并开启sshd服务
    1.7  为了对nginx和sshd的服务进行管理，这里选取supervisor软件包
```bash
sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/'  \
/etc/apk/repositories

apk add --no-cache --virtual .build-deps  gcc libc-dev make openssl-dev \
pcre-dev zlib-dev linux-headers curl jemalloc-dev gd-dev git openssh supervisor
```

## 下载编译nginx ##
1. 下载软件包
```bash
wget -c http://nginx.org/download/nginx-1.10.2.tar.gz \
&& git clone https://github.com/cuber/ngx_http_google_filter_module.git \
&& git clone https://github.comyaoweibinngx_http_substitutions_filter_module.git \
&& git clone https://github.com/aperezdc/ngx-fancyindex.git  \
&& git clone https://github.com/yaoweibin/nginx_upstream_check_module.git
tar -xzf nginx-1.10.2.tar.gz && \
```
2. 编译nginx
```bash
cd nginx-1.10.2/src && \

patch -p1 < ../../nginx_upstream_check_module/check_1.9.2+.patch

cd ..

./configure --prefix=/usr/local/nginx \
--with-pcre \
--with-ipv6 \
--with-http_ssl_module \
--with-http_flv_module \
--with-http_v2_module \
--with-http_realip_module \
--with-http_gzip_static_module \
--with-http_stub_status_module \
--with-http_mp4_module \
--with-http_image_filter_module \
--with-http_addition_module \
--with-http_sub_module  \
--with-http_dav_module  \
--http-client-body-temp-path=/usr/local/nginx/client/ \
--http-proxy-temp-path=/usr/local/nginx/proxy/ \
--http-fastcgi-temp-path=/usr/local/nginx/fcgi/ \
--http-uwsgi-temp-path=/usr/local/nginx/uwsgi \
--http-scgi-temp-path=/usr/local/nginx/scgi \
--add-module=../ngx_http_google_filter_module \
--add-module=../ngx_http_substitutions_filter_module \
--add-module=../ngx-fancyindex \
--add-module=../nginx_upstream_check_module \
--with-ld-opt="-ljemalloc" && \
make -j $(awk '/processor/{i++}END{print i}' /proc/cpuinfo) && make install &&  \
mkdir -p /usr/local/nginx/cache && \
mkdir -p /usr/local/nginx/temp && \
```

## 配置ssh服务
1. 为了使用ssh，需要给root用户设置密码,这里使用chpasswd进行密码的设置
```bash
    echo "root:123456" | chpasswd
```

2. 生成ssh密钥
  2.1  相对其它发行版，alpine的openssh安装后在在配置文件目录/etc/ssh中并没有相关
  的密钥文件模板，需要手动生成.
  ```bash
    /usr/bin/ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key && \
    /usr/bin/ssh-keygen -t dsa -N "" -f /etc/ssh/ssh_host_dsa_key && \
    /usr/bin/ssh-keygen -t ecdsa -N "" -f /etc/ssh/ssh_host_ecdsa_key && \
    /usr/bin/ssh-keygen -t ed25519 -N "" -f /etc/ssh/ssh_host_ed25519_key
  ```
3. ssh允许密码登陆
```bash
    sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/'  /etc/ssh/sshd_config
```


## 配置supervisor ##
1. supervisor的主文件是/etc/supervisor.conf,其主配置文件使用include
   supervisor.d/****.ini包含了子配置文件，因此我们将我们自己的配置文件添加到
   /etc/supervisor.d目录,可以使用docker file的ADD指令添加文件
   ```
    sh -c '[  ! -e "/etc/supervisor.d" ] && mkdir /etc/supervisor.d'
    touch custome.conf
    ```
2. 配置文件custome.conf的内容
2.1  supervisord服务在在docker容器开启的使用运行，为了确保容器在后台一直
运行，需要配置supervisord直接前端运行，不使用daemon方式运行在custome添加如下段:
```bash
    [supervisord]
    nodaemon=true
``` 
2.2  为了使用supervisor管理ssh服务，在custome.conf中添加如下段
```bash
    [program:sshd]
    command=/usr/sbin/sshd -D
```

2.3  为了使用supervisor管理nginx服务，在custome.conf中添加如下段
```bash
    [program:nginx-server]
    command=/usr/local/nginx/sbin/nginx -g 'daemon off;'
```
## 开放端口 ##
1. 在docker容器中需要开放22, 80端口
```bash
    EXPOSE 22
    EXPOSE 80
```

## DOCKERFILE 结构 ##
1. 这个项目dockerfile会使用到的指令有：
```bash
FROM alpine:latest
RUN xxx
ADD custome.conf /etc/supervisor.d/custome.conf
WORKDIR dir
EXPOSE port
CMD ["/usr/bin/supervisord"]
```
2. 项目地址
    
