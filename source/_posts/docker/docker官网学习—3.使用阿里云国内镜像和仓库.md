---
title: docker学习笔记---docker使用阿里云私有仓库
date: 2020-06-29 11:59:58
tags:  docker
categories: docker
comments: true
copyright: true
---



### docker使用阿里云私有仓库



注册阿里云镜像服务:

以下是我的阿里云镜像仓库链接:
https://cr.console.aliyun.com/cn-hangzhou/repositories

一.使用阿里云镜像加速器
https://cr.console.aliyun.com/cn-hangzhou/mirrors

镜像加速地址:

```
https://0w5ygvsg.mirror.aliyuncs.com
```
如果是Centos系统,可以通过修改daemon配置文件/etc/docker/daemon.json来使用加速器:

```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://0w5ygvsg.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```
----

<!--more-->

#### 使用阿里云的镜像仓库

首先在阿里云镜像服务控制台创建镜像仓库和命令空间:

https://cr.console.aliyun.com/cn-hangzhou/repositories

我的镜像仓库和命名空间都是:jesse_images
这是我的镜像仓库地址:registry.cn-hangzhou.aliyuncs.com/jesse_images/jesse_images


下面演示,如何推送本地镜像到阿里云仓库

1.在本地docker服务器登陆阿里云镜像仓库

```
[root@localhost docker_python]$docker login --username=jessehuang408 registry.cn-hangzhou.aliyuncs.com
```

2.将本地镜像推送到仓库执行以下两条命令

- 为本地镜像打个标签

```
docker tag [ImageId] registry.cn-hangzhou.aliyuncs.com/jesse_images/jesse_images:[镜像版本号]
```


- 将镜像推送到仓库

```
docker push registry.cn-hangzhou.aliyuncs.com/jesse_images/jesse_images:[镜像版本号]
```

下面演示将friendlyhello这个镜像推送到阿里云远程仓库

```
root@localhost docker_python]$docker images
REPOSITORY                                                    TAG                 IMAGE ID            CREATED             SIZE
friendlyhello                                                 latest              f091d1bb803c        43 minutes ago      131MB
```

```
docker tag f091d1bb803c registry.cn-hangzhou.aliyuncs.com/jesse_images/jesse_images:v2.0

docker push registry.cn-hangzhou.aliyuncs.com/jesse_images/jesse_images:v2.0

```
在本机可以看到镜像:

```
root@localhost docker_python]$docker images
REPOSITORY                                                    TAG                 IMAGE ID            CREATED             SIZE
friendlyhello                                                 latest              f091d1bb803c        43 minutes ago      131MB
registry.cn-hangzhou.aliyuncs.com/jesse_images/jesse_images   v2.0                f091d1bb803c        43 minutes ago      131MB

```

登陆阿里云的镜像服务控制台,在镜像仓库的管理界面可以看到上传上去的镜像

如果是从阿里云镜像仓库拉取镜像,执行以下命令:

```
sudo docker pull registry.cn-hangzhou.aliyuncs.com/jesse_images/jesse_images:[镜像版本号]
```
