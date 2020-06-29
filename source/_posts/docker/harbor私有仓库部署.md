---
title: harbor私有仓库部署
date: 2020-06-26 11:59:58
tags:  docker
categories: docker
comments: true
copyright: true
---



## harbor私有仓库部署

### 介绍

harbor是docker的私有仓库,可以部署在局域网服务器上,用来管理docker镜像.虽然docker官方也提供公共镜像仓库,但是由于是境外网站,拉取镜像速度非常慢,而且有被墙的可能.部署私有仓库非常有必要

harbor是vmware公司开源的企业级的docker registry管理项目.

> 处于数据脱敏需要,以下内容中隐藏了真实域名.而使用hub.xxxxxx.com替代

---

### 社区

harbor github: [goharbor/harbor](https://github.com/goharbor/harbor)

官网文档介绍: [harbor doc](https://goharbor.io/docs/1.10/)

在部署中遇到的各种坑,都可以通过查阅文档,或者搜索github的issue解决

---

### 框架

harbor是docker-compose部署的.包括一系列组件:nginx.core,log,register等等.

但是由于本机已经存在一个Nginx镜像.所以用Nginx代理到harbor的Nginx.**如果是独立的服务器部署Harbor的话,则不会存在这个问题,可以直接跳过这一章节.**

nginx代理框架大概是:

**nginx---->harbor-nginx----->habor**

由于docker提交镜像需要Https协议,所以:

**nginx---301跳转到nginx https----->harbor-nginx http----> habor**

但是这样的部署方式,有一个问题:

私有仓库可以正常login但是push镜像的时候,又提示未验证.

该问题尝试过很多解决方案,但是均无法解决

<!--more-->

```
[root@idc-function-docker ~]# docker login --username=admin -pHarbor12345 hub.xxxxxx.com
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
[root@idc-function-docker ~]# docker push hub.xxxxxx.com/master/nginx:latest
The push refers to repository [hub.xxxxxx.com/master/nginx]
d37eecb5b769: Pushing [==================================================>]  3.584kB
99134ec7f247: Preparing
c3a984abe8a8: Preparing
unauthorized: authentication required
```

所以现在的架构是

**nginx---301跳转到Nginx https------> harbor-nginx https------>harbor**

---

### harbor部署前提条件

* 安装docker-ce
* 安装docker-composer
* 准备一个空目录.比如/data.或者/data/harbor (注意,最好是空目录,不要和其他项目混杂一起)
* 安装好https域名证书

---

### 具体步骤

docker和docker-compose的安装就略过了.这里提一句,我用的是acme.sh部署letsencrypt的证书

接下来开始部署harbor

* 1.去github下载离线安装包.离线安装包虽然比较大,但是安装过程快速,且不会中断

这里安装的是最新版,v1.10.1.

```
在github下载 harbor-offline-installer-v1.10.1.tgz
tar xvf 解压
```

* 2.解压后,进入harbor文件,编辑harbor.yaml配置文件.需要改动以下几个地方

```
hostname hub.xxxxxx.com  #定义主机名,可以使用IP,也可以用域名

#定义https的服务器证书和秘钥的文件路径
https:
  # https port for harbor, default is 443
  port: 443
  # The path of cert and key files for nginx
  certificate: /data/letsencrypt/hub.xxxxxx.com/fullchain.cer
  private_key: /data/letsencrypt/hub.xxxxxx.com/hub.xxxxxx.com.key
  
#这是harbor命令行和浏览器登陆的初始密码..用户名是admin
harbor_admin_password: Harbor12345

#这是数据目录,最好是一个空目录
data_volume: /data/apps/harbor

#日志文件保存路径
log:
    location: /data/logs/harbor
    
```

* 3.执行install.sh文件.

```
#该脚本会检查主机的环境,拉取相关镜像.以及根据配置文件生成docker-compose文件.
[root@idc-function-docker harbor]#./install.sh 
```

>  这个脚本执行到最后会报错,提示80端口和nginx容器已经被占用了.但是没关系.

* 4.修改docker-composer文件.(如果本机上没有nginx或者80端口没有被占用,这一步可以不做)

```
#将Nginx容器名改成hub-nginx
#将80和443的端口映射修改一下,比如我这里
proxy:
    image: goharbor/nginx-photon:v1.10.1
    container_name: hub-nginx
ports:
      - 8081:8080
      - 4443:8443
```

* 5. 启动docker-compose

```
[root@idc-function-docker harbor]#docker-compose up -d
```

* 6. 宿主机的nginx开启代理. (如果不是采用nginx代理的话,可以忽略这一步)

```
#我这里是有一个独立的Nginx容器
[root@idc-function-docker harbor]# cat/data/conf/nginx/conf.d/dwd-docker-hub.conf



server {

  server_name hub.xxxxxx.com;
  listen 443 ssl http2;
  ssl_certificate  /data/letsencrypt/hub.xxxxxx.com/fullchain.cer;
  ssl_certificate_key /data/letsencrypt/hub.xxxxxx.com/hub.xxxxxx.com.key;
  include /data/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
  ssl_dhparam /data/letsencrypt/ssl-dhparams.pem; # managed by Certbot

  add_header      Strict-Transport-Security "max-age=31536000" always;
  location / {

    proxy_pass https://172.16.20.30:4443; #代理到宿主机的4443端口,宿主机会将4443代理到hub-docer容器的443端口
    client_max_body_size 2000m; #这里要定义大一点,否则提交镜像的时候,会提示 413 Request Entity Too Large
                proxy_buffering off;
                proxy_ssl_verify off;
                proxy_set_header Host $http_host;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "Upgrade";
                proxy_set_header X-Forwarded-Proto $scheme;
  }

}

server {
    if ($host = hub.xxxxxx.com){
        return 301 https://$host$request_uri;
    } # managed by Certbot


  listen 80;
    server_name hub.xxxxxx.com;
    return 404; # managed by Certbot


}
```

至此,部署就完成了.

我之前在部署的过程中遇到无数的坑,,可能都是由于harbor没有使用Https引起的

---

浏览器访问https://hub.xxxxxx.com,用初始的账号密码登陆,可以新建项目,创建用户等.

我这里创建了一个master的公开项目

---



登陆和push镜像没有任何问题

```
root@idc-function-docker ~]# docker login --username=admin -pHarbor12345 hub.xxxxxx.com
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
[root@idc-function-docker ~]# docker push hub.xxxxxx.com/master/nginx:latest
The push refers to repository [hub.xxxxxx.com/master/nginx]
d37eecb5b769: Layer already exists
99134ec7f247: Layer already exists
c3a984abe8a8: Layer already exists
latest: digest: sha256:7ac7819e1523911399b798309025935a9968b277d86d50e5255465d6592c0266 size: 948
[root@idc-function-docker ~]#
```

在另外一台客户端上,尝试pull镜像不需要密码.如果需要密码pull镜像,可以将项目设置为私有

```
[root@localhost ~]$docker pull  hub.xxxxxx.com/master/nginx:latest
latest: Pulling from master/nginx
Digest: sha256:7ac7819e1523911399b798309025935a9968b277d86d50e5255465d6592c0266
Status: Downloaded newer image for hub.xxxxxx.com/master/nginx:latest
```

