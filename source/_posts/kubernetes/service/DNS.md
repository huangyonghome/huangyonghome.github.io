---
title: Kubernetes DNS
date: 2021-02-06 23:39:58
tags: k8s
categories: [kubernetes,Service]
comments: true
copyright: true
---



## DNS介绍

### 介绍

kubernets的所有资源.包括Service,Pod都有生命周期,会频繁的销毁和创建.这些资源的IP地址也会随之动态变化.所以Kubernetes使用DNS实现通过资源名解析IP地址.



### DNS服务器

Kubernetes集群安装了默认的Core-dns组件(通过Pod方式运行).以及kube-dns的service.

```
[root@k8s-master ~]$kubectl get pods -n kube-system | grep dns
coredns-7f9c544f75-9sh28                   1/1     Running   2          324d
coredns-7f9c544f75-jgmqq                   1/1     Running   2          324d

#下方这个10.96.0.10就是kubernetes集群的内部DNS服务器
[root@k8s-master ~]$kubectl get svc -n kube-system | grep dns
kube-dns         ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP,9153/TCP   324d 
```

<!--more-->

### pod容器内部dns解析

创建一个临时的pod容器,测试DNS解析效果.下面的命令临时运行了一个busybox的镜像

```
[root@k8s-master ~]$kubectl run -it dns-test --rm --image=busybox:1.28.4 -- sh
```

> 不要使用latest版本的镜像,其dns解析有问题.最好使用1.28.4版本的

下方是Pod容器的内部dns服务器信息

```
/ # cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local localdomain
options ndots:5
```

`resolv.conf` 配置文件说明

**nameserver**：指明DNS服务器地址.也就是上文提到的`kube-dns` 的service

**search**：当原始域名不能被DNS解析时，resolver会将该域名加上search指定的参数，重新请求DNS，直到被正确解析或试完search指定的列表为止 options：dns配置 

**ndots:5**：所有DNS查询中，如果“.”的个数少于5个，则会根据search中配置的列表依次在对应域中先进行搜索，如果没有返回，则最后再直接查询域名本身

为了了解`search`和`ndots` 的概念,我们先要了解FQDN的概念.`FQDN(Fully qualified domain name)`即完整域名。一般来说如果一个域名以`.`结束，就表示一个完整域名。比如`www.abc.xyz.`就是一个`FQDN`，而`www.abc.xyz`则不是`FQDN`。了解了这个概念之后我们就来看`search`和`options ndots`。

如果一个域名是`FQDN`，那么这个域名会被转发给DNS服务器进行解析。如果域名不是`FQDN`，那么这个域名会到`search`搜索解析，还是通过一个例子说明，比如访问`abc.xyz`这个域名，因为它并不是一个`FQDN`，所以它会和`search`域中的值进行组合而变成一个`FQDN`，以上文的`resolv.conf`为例，这域名会这样组合：

```
abx.xyz.default.svc.cluster.local.
abc.xyz.svc.cluster.local.
abc.xyz.cluster.local.
```

然后这些域名先被`kube-DNS`解析，如果没有解析成功再由宿主机的`DNS`服务器进行解析。

而`ndots`是用来表示一个域名中`.`的个数在不小于该值的情况下会被认为是一个`FQDN`。简单说这个属性用来判断一个不是以`.`结束的域名在什么条件下会被认定为是一个`FQDN`.上面的示例中ndots为5,也就是说如果一个域名中`.`的数量大于等于5，即使域名不是以`.`结尾，也会被认定为是一个`FQDN`。比如：域名是`abc.xyz.xxx.yyy.zzz.aaa`这个域名就是`FQDN`.

之所以会有`search`域主要还是为了方便k8s内部服务之间的访问。比如：k8s在同一个`namespace`下是可以直接通过服务名称进行访问的，其原理就是会在`search`域查找，比如上面的`resolv.conf`中`jplat、oms-dev`着两个其实都是这两个pod所在的`namespace`的名称。所以通过服务名称访问的时候，会和`search`域进行组合，这样最终域名会组合成`servicename.namespace.svc.cluster.local`。而如果是跨`namespace`访问，则可以通过`servicename.namespace`这样的形式，在通过和`search`域组合，依然可以得到`servicename.namespace.svc.cluster.local`。



### DNS解析

#### 解析对象

Kubernetes集群中的每个Service资源都会被指派一个DNS名称.客户端Pod的DNS搜索列表默认是搜索自己的`namespace` 名称空间内的资源.

例如上文的`resolv.conf` 文件内的search搜索列表为`search default.svc.cluster.local svc.cluster.local cluster.local localdomain` .此时Pod可以直接搜索`default` 名称空间下的所有Service:

例如.使用上面的临时busybox容器解析`my-svc`的Service

```
/ # nslookup my-svc
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      my-svc
Address 1: 10.96.51.58 my-svc.default.svc.cluster.local
```

上面的IP地址`10.96.51.58` 表示成功解析到该Service的IP.`my-svc.default.svc.cluster.local`这个就是该Service的FQDN完全限定域名.

其中:

**`default`** ---表示名称空间,我们的名称空间名字就是默认的default

**`svc`** ----------------表示资源类型,这里是Service

**`cluster.local`** --k8s集群域名

也可以解析其他名称空间内的资源,比如解析`kube-system` 名称空间下的DNS服务器的Service.(DNS服务器本身也会被指定一个DNS名称).就可以通过`<svc-name>.<namespace-name>` 实现.比如下面解析`kube-system`名称空间下的`kube-dns`的Service

```
/ # nslookup kube-dns.kube-system
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kube-dns.kube-system
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
```

实际上DNS解析的是完全FQDN域名,只不过后面一部分内容`default.svc.cluster.local` 可以省略罢了.默认就是解析当前名称空间下的资源

```
/ # nslookup my-svc
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      my-svc
Address 1: 10.96.51.58 my-svc.default.svc.cluster.local

#等同于:
/ # nslookup my-svc.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      my-svc.default.svc.cluster.local
Address 1: 10.96.51.58 my-svc.default.svc.cluster.local
```

在kubernetes官网中也提到:

假设在 Kubernetes 集群的名字空间 `bar` 中，定义了一个服务 `foo`。 运行在名字空间 `bar` 中的 Pod 可以简单地通过 DNS 查询 `foo` 来找到该服务。 运行在名字空间 `quux` 中的 Pod 可以通过 DNS 查询 `foo.bar` 找到该服务。



#### Service A记录

对于普通的Service资源.会以`<service-name>.<namespace-name>.svc.cluster.local`这种形式被分配一个DNS A记录.也就是上文中的`my-svc`的`10.96.51.58`这个IP地址.

如果是对于无头服务(headless service).这种service没有IP.但是也会以上面的形式被指派一个DNS的A记录.只不过这种记录和普通Service不同,而是被解析成对应服务的POD集合的Pod的IP.客户端使用标准的负载均衡策略从这组Pod中进行选择.

例如下面创建一个headless的svc.和普通svc的区别在于`clusterIP的值为None`.

```
[root@k8s-master ~]$cat deployment-kubia-v1.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hsq1
  labels:
     app: hsq-openapi
spec:
  containers:
    - name: nginx
      image: nginx

---
apiVersion: v1
kind: Pod
metadata:
  name: hsq2
  labels:
     app: hsq-openapi
spec:
    containers:
       - name: nginx
         image: nginx
---
#一个yaml文件可以定义多种资源,中间用---隔开

apiVersion: v1
kind: Service
metadata:
  name: hsq-openapi
spec:
  selector:
    app: hsq-openapi
  clusterIP: None
  ports:
  - port: 80
    targetPort: 80
```

> headless服务一般用于statefulset资源.不能用于deployment控制器



创建该文件后查看`hsq-openapi`service:

```
[root@k8s-master ~]$kubectl describe svc hsq-openapi
Name:              hsq-openapi
Namespace:         default
Labels:            <none>
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"hsq-openapi","namespace":"default"},"spec":{"clusterIP":"None","p...
Selector:          app=hsq-openapi
Type:              ClusterIP
IP:                None
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.100.36.66:80,10.100.36.69:80  #这里是后端pod列表
Session Affinity:  None
Events:            <none>
```



**headless类型服务的DNS解析**

仍然使用上文中的busybox测试容器.解析`hsq-openapi` service 的A记录.可以看到解析的结果返回了2个pod的IP地址列表.对于这种类型的service.和普通的service不同.他解析出来的是POD的ip地址列表

```
/ # nslookup hsq-openapi
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      hsq-openapi
Address 1: 10.100.36.69 10-100-36-69.hsq-openapi.default.svc.cluster.local
Address 2: 10.100.36.66 10-100-36-66.hsq-openapi.default.svc.cluster.local
/ # ???
```

---

#### Pod的A记录

一般而言,Pod会对应如下DNS名字解析: `pod-ip-address.<namespace-name>.pod.cluster.local` 例如对于上面例子中的iP为`10.100.36.69` 的Pod.对应的DNS名称为:

```
/ # nslookup 10-100-36-69.default.pod.cluster.local  #DNS名称
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      10-100-36-69.default.pod.cluster.local
Address 1: 10.100.36.69 10-100-36-69.hsq-openapi.default.svc.cluster.local
```



### k8s默认的DNS策略

k8s提供了5种DNS策略，如下：

- `Default`: Pod 从运行所在的节点继承名称解析配置。
- `ClusterFirst`: 与配置的集群域后缀不匹配的任何 DNS 查询（例如 “[www.kubernetes.io](https://links.jianshu.com/go?to=http%3A%2F%2Fwww.kubernetes.io)”） 都将转发到从节点继承的上游名称服务器。集群管理员可能配置了额外的存根域和上游 DNS 服务器。
- `ClusterFirstWithHostNet`：对于以 hostNetwork 方式运行的 Pod，应显式设置其 DNS 策略 `ClusterFirstWithHostNet`。
- `None`: 此设置允许 Pod 忽略 Kubernetes 环境中的 DNS 设置。Pod 会使用其 `dnsConfig` 字段 所提供的 DNS 设置。

k8s默认使用的DNS策略是`ClusterFirst`，这点需要注意，也就是说域名解析会优先使用集群的DNS（`kube-DNS`）进行查询，如果k8s的DNS解析失败，会转发到宿主机的DNS进行解析。



