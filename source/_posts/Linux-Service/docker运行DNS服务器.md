---
title: docker运行DNS服务器
date: 2018-08-14 09:59:58
tags: DNS
categories: [Linux-Service ]
comments: true
copyright: true
---



## docker运行DNS服务器

#### 概述

linux提供了Bind工具搭建DNS服务,但是bind配置太过复杂,需要配置正向解析,zone区域等.而且很多功能是完全用不上的.所以这里选择使用dnsmasq来作为内部DNS服务器.

dnsmasq非常小巧,简单,配置十分方便.只有3个配置文件:

<!--more-->

- resolve.dnsmasq : 这个配置文件定义了上游公网的DNS服务器地址.dnsmasq把DNS请求转发给公网DNS解析
- dnsmasqhosts: 静态绑定IP和域名,语法和/etc/hosts一样.这是内部DNS的主要配置文件,用于自定义域名解析
- dnsmasq.conf: 这个是dnsmasq的配置文件,这个配置只有2条语句,定义2个字段指向上面2个配置文件.

#### 配置步骤

* 拉取dnsmasq系统镜像

  ```
  docker pull andyshinn/dnsmasq
  ```

* 新建配置文件路径,定义配置文件

  ```
  mkdir -pv /data/conf/dns
  cd /data/conf/dns
  ```

  - 定义resolv.dnsmasq文件

    ```
    vi resolv.dnsmasq
    nameserver 202.96.209.133
    nameserver 114.114.114.114
    ```

  - 定义dnsmasqhosts

    ```
    vi dnsmasqhosts
    10.0.4.230 www.test.com
    10.0.4.231 www.jesse.com
    ```

  - 定义dnsmasq.conf配置文件

    ```
    vi dnsmasq.conf
    resolv-file=/etc/resolv.dnsmasq
    addn-hosts=/etc/dnsmasqhosts
    ```

    

* 定义docker启动文件

  ```
  version: "2"
  services:
    docker-dns:
      container_name: dns
      image: andyshinn/dnsmasq
      hostname: dns
      volumes:
        - /data/conf/dns/resolv.dnsmasq:/etc/resolv.dnsmasq
        - /data/conf/dns/dnsmasqhosts/:/etc/dnsmasqhosts
        - /data/conf/dns/dnsmasq.conf/:/etc/dnsmasq.conf
        - /etc/localtime:/etc/localtime:ro
      ports:
        - 53:53/tcp
        - 53:53/udp
      cap_add:
        - NET_ADMIN
      restart: on-failure:1
  ```



* 安装compose(如果已经安装,则可以跳过此步骤)

  ```
  wget  https://github.com/docker/compose/releases/download/1.22.0/docker-compose-Linux-x86_64
  sudo mv docker-compose-Linux-x86_64 /usr/local/bin/docker-compose
  sudo chmod +x  /usr/local/bin/docker-compose
  
  [work@docker ~]$docker-compose --version
  docker-compose version 1.22.0, build f46880fe
  ```

  

* 运行容器

  ```
  docker-compose -f /data/conf/dns/dns.yaml up -d
  ```

* 进入容器

  ```
  [work@docker ~]$docker exec -it dns /bin/sh
  ```

  > 注意: 不能用/bin/bash进入容器

```
[work@docker ~]$docker exec -it dns /bin/bash
rpc error: code = 2 desc = oci runtime error: exec failed: container_linux.go:247: starting container process caused "exec: \"/bin/bash\": stat /bin/bash: no such file or directory"
```

