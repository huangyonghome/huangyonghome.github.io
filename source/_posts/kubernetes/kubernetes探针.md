---
title: kubernets探针
date: 2020-06-26 11:59:58
tags: k8s
categories: kubernetes
comments: true
copyright: true
---



## kubernets探针

## 介绍

kubernetes一共有2种探针:

**存活探针**

**就绪探针**

---

## 就绪探针

### 就绪探针工作介绍

就绪探针会定期调用,检查特定的pod是否准备就绪接收客户端的请求.当容器启动时,会等待一个时间,然后执行第一次准备就绪检查.如果某个Pod没有通过探针探测,则会从服务中删除该pod,如果pod再次准备就绪,则会重新添加到Service

就绪探针确保客户端只与正常的Pod交互.

---

### 就绪探针的类型

* Exec探针 . 使用command命令,如果命令结果返回0,则说明pod准备就绪.
* HTTP GET探针. 向容器发送HTTP GET.通过响应状态码判断pod容器是否准备好
* TCP socket探针.尝试连接Pod容器的TCP端口.判断pod容器的某个端口是否正常工作

---

<!--more-->

### 向pod容器添加探针

编辑kubia的rc配置文件,在spec.containers下添加readinessProbe配置

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia
spec:
  replicas: 4
  selector:
      app: kubia
  template:
    metadata:
      labels:
        app: kubia

    spec:
      containers:
        - name: kubia
          image: luksa/kubia
          ports:
          - containerPort: 8080
          readinessProbe:
            exec: #存活探针类型
              command: #探针命令
                - ls
                - /var/ready
                
```
> 重新编辑rc文件后,存活探针并不会对已经存在的Pod生效.删除所有pod,等待rc重新创建

让yaml文件生效

```
[root@k8s-master ~]# kubectl apply -f kubia-rc.yaml
replicationcontroller/kubia unchanged
```
删除Pods,重新创建后,查看pods的状态

```
[root@k8s-master ~]# kubectl get pods
NAME                READY   STATUS    RESTARTS   AGE
kubia-6gvhg         0/1     Running   0          61s
kubia-qlpdt         0/1     Running   0          61s
kubia-w8c82         0/1     Running   0          61s
kubia-w9spj         0/1     Running   0          61s

```
pods的READY是0.表示pod容器虽然正在运行中,但是未准备就绪

向其中一个pod容器创建/var/ready文件

```
[root@k8s-master ~]# kubectl exec kubia-6gvhg -- touch /var/ready
```

但是容器并不会马上就绪.这是因为pod默认每隔10s探测一次.通过```kubectl describe pod kubia-6gvhg```可以看到就绪探针的策略

```
[root@k8s-master ~]# kubectl describe pods kubia-6gvhg

Readiness:      exec [ls /var/ready] delay=0s timeout=1s period=10s #success=1 #failure=3
Warning  Unhealthy  2m53s (x30 over 7m43s)  kubelet, k8s-node2  Readiness probe failed: ls: cannot access /var/ready: No such file or directory
```

10s后,可以看到就只有一个pod准备就绪

```
[root@k8s-master ~]# kubectl get pods
NAME                READY   STATUS    RESTARTS   AGE
kubia-6gvhg         1/1     Running   0          9m45s
kubia-qlpdt         0/1     Running   0          9m45s
kubia-w8c82         0/1     Running   0          9m45s
kubia-w9spj         0/1     Running   0          9m45s
```
通过测试发现,客户端的请求永远只转发到已经准备就绪的Pod容器内,而其他容器则不接受请求

```
[work@k8s-node1 ~]$ while true;do curl http://10.96.170.37;sleep 1;done
You've hit kubia-6gvhg
You've hit kubia-6gvhg
You've hit kubia-6gvhg
You've hit kubia-6gvhg
You've hit kubia-6gvhg
You've hit kubia-6gvhg
You've hit kubia-6gvhg
```

其他两种探测类型的是否方法也类似

---



