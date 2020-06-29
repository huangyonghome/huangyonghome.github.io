---
title: k8s挂载NFS网络磁盘
date: 2020-06-26 11:59:58
tags: k8s
categories: kubernetes
comments: true
copyright: true
---



## k8s挂载NFS网络磁盘

按照<kubernetes in action>这本书NFS做持久化存储的例子,发现了一个坑.,启动pod失败,报如下错误

```
chown: changing ownership of '/data/db': Operation not permitted
```

网上也有人遇到这个问题.可以参考这篇文档: [Kubernetes 集群挂载NFS Volume](https://blog.csdn.net/herhun_chen/article/details/90247123)

<!--more-->

以下是pod的yaml文件:

```
[root@k8s-m01 ~]# cat mongodb-pod-nfs.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  volumes:
     - name: mongodb-data
       nfs:
         server: 10.111.5.184
         path: /data/k8s/

  containers:
     - image: mongo
       name: mongodb
       volumeMounts:
         - name: mongodb-data
           mountPath: /data/db
       ports:
         - containerPort: 27017
           protocol: TCP
```



出现这种`Operation not permitted`的权限类问题,肯定是NFS的挂载有问题.但是在所有k8s的节点上往NFS共享磁盘写文件,又是正常的.



解决这个问题需要在NFS服务器的`/etx/exports`配置文件修改成如下配置

```
[work@hsq-beta-rpc ~]$cat /etc/exports
/data/k8s/ 10.111.0.0/16(rw,fsid=0,async,no_subtree_check,no_auth_nlm,insecure,no_root_squash)
```

> 注意,以上配置其实是一行.没有换行

