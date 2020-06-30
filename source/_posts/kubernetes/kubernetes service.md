---
title: kubernetes Service
date: 2020-06-26 11:59:58
tags: k8s
categories: kubernetes
comments: true
copyright: true
---



## kubernetes Service

## 介绍

Kubemetes服务(Service)是一种为一组功能相同的pod 提供单一不变的接入点的资源.客户端通过Service的IP 地址和端口号建立连接.

这些连接会被路由到提供该服务的任意一个pod上。 通过这种方式，客户端不需要知道每个单独的提供服务的pod的地址，这样这些pod就可以在集群中随时被创建或移除。

举个例子，考虑一个图片处理backend，它运行了3个副本。这些副本是可互换的——frontend不需要关心它们调用了哪个backend副本。然而组成这一组backend 程序的 Pod 实际上可能会发生变化，frontend客户端不应该也没必要知道，而且也不需要跟踪这一组 backend 的状态。Service定义的抽象能够解耦这种关联。

---

### 定义service

* 通过`kubectl expose` 命令手动创建

例如为kubia这个replicaSet创建一个Service:

```
[root@k8s-master ~]# kubectl get rc
NAME    DESIRED   CURRENT   READY   AGE
kubia   4         4         4       2d9h
[root@k8s-master ~]# kubectl expose rc kubia --port=80 --target-port=8080 --name kubia-nginx
service/kubia-nginx exposed
```

kubectl expose 创建Service命令格式:

<!--more-->

```
kubectl expose rc名 --port=源端口 --target-port=pod容器端口 --name Service名
```

* 通过yaml文件创建.

下面是一个yaml文件的例子,还是实现上面的功能,为Kubia这个rc创建Service

```
apiVersion: v1 #Service属于v1这个版本
kind: Service
metadata: #Service的元数据信息
  name: kubia
spec:
#selector标签选择器,选择所有后端有app标签且值为kubia的pod
  selector: 
     app: kubia
#新版的k8s中ports的值是一个对象.其中port是Service的源端口
#targetPort是pod容器端口.如果Pod容器为端口定义了名称,则这里可以不指定具体端口号,而使用名称代替
  ports:
   - port: 80
     targetPort: 8080
```
接下来使用`kubectl create`命令创建

```
[root@k8s-master ~]# kubectl create -f kubia-svc-new.yaml
service/kubia created
[root@k8s-master ~]# kubectl get svc
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes    ClusterIP   10.96.0.1      <none>        443/TCP   20d
kubia         ClusterIP   10.96.170.37   <none>        80/TCP    4s
kubia-nginx   ClusterIP   10.96.154.64   <none>        80/TCP    8m54s
[root@k8s-master ~]#
```
此时可以看到kubia-nginx服务已被创建,并且获得了一个CLUSTER-IP地址.  
随便进入到一个Pod容器内部或者随便在一个k8s集群节点服务器上,试图访问该service

> 注意,此时这个Service仍然只能由Pod容器内部和k8s集群节点访问.外部客户端无法访问

```
[work@k8s-node1 ~]$ while true;do curl http://10.96.170.37;sleep 1;done
You've hit kubia-pdpg8
You've hit kubia-wc5wg
You've hit kubia-jw884
You've hit kubia-pdpg8
You've hit kubia-pdpg8
You've hit kubia-jw884
You've hit kubia-wc5wg
You've hit kubia-pdpg8
```
可以看到Service是负载均衡方式,平均分担流量到后端的pod中.

Sevice的负载均衡策略默认是None.也就是轮询方式负载流量,可以修改ClientIP,即将同一个IP客户端的访问流量,固定到一个pod节点上

这里就涉及到修改现有Service策略的2种办法:

* 1. `kubectl edit svc service名`

```
[root@k8s-master ~]# kubectl edit svc kubia

#弹出以下Service配置文件

apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2020-04-04T00:58:21Z"
  name: kubia
  namespace: default
  resourceVersion: "4209349"
  selfLink: /api/v1/namespaces/default/services/kubia
  uid: 18020179-8d72-4be2-b64d-610b1dda8a44
spec:
  clusterIP: 10.96.170.37
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: kubia
  sessionAffinity: None #修改为ClientIP
  type: ClusterIP
status:
  loadBalancer: {}

#保存文件,退出.提示:
service/kubia edited
```

此时客户端的请求会固定到某一个Pod中

```
You've hit kubia-pdpg8
You've hit kubia-wc5wg
You've hit kubia-l7t82
You've hit kubia-wc5wg
You've hit kubia-pdpg8
You've hit kubia-pdpg8
You've hit kubia-pdpg8
You've hit kubia-pdpg8
You've hit kubia-pdpg8
You've hit kubia-pdpg8
You've hit kubia-pdpg8
You've hit kubia-pdpg8
You've hit kubia-pdpg8
You've hit kubia-pdpg8
You've hit kubia-pdpg8
You've hit kubia-pdpg8
```

* 2.通过PATCH命令,打补丁方式修改

命令格式: `kubectl patch svc service名 -p '{"键名":{"子键名":"值"}}' `

例如,将spec配置下的sessionAffinity重新修改为None

```
[root@k8s-master ~]# kubectl patch svc kubia -p '{"spec":{"sessionAffinity":"None"}}'
service/kubia patched

```
此时客户端的请求会立即生效,变回负载均衡


```
[work@k8s-node1 ~]$ while true;do curl http://10.96.170.37;sleep 1;done
You've hit kubia-pdpg8
You've hit kubia-pdpg8
You've hit kubia-pdpg8
You've hit kubia-jw884
You've hit kubia-jw884
You've hit kubia-wc5wg
```
> Service只支持2种流量转发方式:None和ClientIP.因为Service是工作在4层(TCP,UDP端口转发),所以不支持cookie等http特性

> 以上2种方式,无论哪种都只在当前生效,而不会修改我们创建Service编写的yaml文件.

---

### 多个端口Service

一个Service可以映射多个端口,而无需创建多个服务.但是当映射多个端口时,必须要给每个端口指定名字.例如下面的例子

```
apiVersion: v1
kind: Service
metadata:
  - name: kubia

spec:
  slector:
    app: kubia
  
  ports:
    - name: http
      port: 80
      targetPort: 8080
      
    - name: https
      port: 443
      targetPort: 8443
```

---

### Service环境变量

kubectl在初始化pods容器时会为Pod容器定义一系列的变量.由于我刚才的Service后于Pods容器创建,所以删除Pod容器,然后查看新pod容器的环境变量
```
[root@k8s-master ~]# kubectl delete pods --all

[root@k8s-master ~]# kubectl exec kubia-fcc59 env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=kubia-fcc59
KUBIA_PORT_80_TCP=tcp://10.96.170.37:80
KUBIA_PORT_80_TCP_PORT=80
KUBIA_NGINX_PORT_80_TCP_PORT=80
KUBIA_NGINX_PORT_80_TCP_PROTO=tcp
KUBIA_SERVICE_HOST=10.96.170.37     #kubia的Service地址
KUBIA_PORT_80_TCP_ADDR=10.96.170.37 #kubia的Service端口

```
所以可以在Pod容器内部使用环境变量访问Service

```
root@kubia-fcc59:/# curl  http://$KUBIA_SERVICE_HOST
You've hit kubia-hd429
```
---

### Service DNS

k8s集群维护着一个默认的DNS服务器.而pods如果没有显示指定的话,默认使用的是kubenetes的默认DNS服务器:

DNS服务器在k8s的系统名称空间下:
```
[root@k8s-master ~]# kubectl get pods --namespace=kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
coredns-7f9c544f75-9sh28                   1/1     Running   1          20d
coredns-7f9c544f75-jgmqq                   1/1     Running   1          20d
```
所有集群的Service资源共享同一个后缀FQDN名:

**svc.cluster.local**

完整的Service服务的FQDN名为:

**ServiceName.Namespace.svc.cluster.local.**

而默认情况下Service等资源的名称空间为default.所以在pods内部可以通过如下的FQDN访问Service

**kubia.default.svc.cluster.local**

```
root@kubia-fcc59:/# curl http://kubia.default.svc.cluster.local
You've hit kubia-hd429
```
如果pods容器和Service服务在同一个名称空间下,甚至可以省略default.svc.cluster.local后缀.而只使用ServiceName服务名来访问.这非常方便

```
root@kubia-fcc59:/# curl http://kubia
You've hit kubia-xgnkq
root@kubia-fcc59:/# curl http://kubia.default
You've hit kubia-rm9qf

#如果查看pods的resolv.conf文件,可以看到default.svc.cluster.local以及svc.cluster.local补全域名

root@kubia-fcc59:/# cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local localdomain
options ndots:5

此时pod容器访问kubia.会自动补全FQDN域名变为:
kubia.default.svc.cluster.local
```

> 注意无论是DNS还是IP,都无法从任何地方去使用ping命令连接Service.这是因为Serivce的IP是一个虚拟IP.而且需要通过IP和Port结合才有效,无法单独访问Service IP.

---

### ExternalName类型服务

ExternalName类型服务将一个外部服务的域名做一个CNAME域名解析.使集群内部的Pod容器通过访问ExternalName类型的服务名访问外部服务

因为ExternalName类型的服务仅仅在DNS层面做Cname接,所以该类型的服务不会获得IP地址.并且ExternalName类型的服务需要指定外部服务的FQDN完全合格域名,而非具体的IP地址

下面是一个ExternalName类型服务的简单例子

```
apiVersion: v1
kind: Service
metadata:
  name: kubia-svc-externalname
spec
  tye: ExternalName
  externalName: someapi.somecompany.com
  ports:
    - port: 80

```

### 从集群外部访问Service

有两种类型服务可以使集群外部客户端访问Service:

* NodePort
* LoadBalancer

> 此外还可以创建一个ingress资源.这是和Service完全不同的机制,以后再学习

#### NodePort

顾名思义,节点端口,在每个节点上将打开同一个端口.外部客户端可以访问节点上该端口.节点将会将流量代理到Service

如果设置Service的type的值为"NodePort",Kubernetes master将从给定的配置范围内（默认：30000-32767)分配端口，每个 Node 将从该端口（每个 Node 上的同一端口）代理到 Service。该端口将通过 Service 的 spec.ports[*].nodePort 字段被指定。

如果需要指定的端口号，可以配置 nodePort 的值，系统将分配这个端口，否则调用 API 将会失败（比如，需要关心端口冲突的可能性）。

下面创建一个NodePort的yaml文件

```
apiVersion: v1
kind: Service
metadata:
  name: kubia-svc-nodeport

spec:
  type: NodePort  #定义Service的type为NodePort
  selector:
       app: kubia

  ports:
    - port: 80
      targetPort: 8080
      nodePort: 32013 #k8s节点服务器上指定的端口

```

可以看到启动了个Kubia-svc-nodeport的服务,type是Nodeport.

```
[root@k8s-master ~]# kubectl get svc
NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes           ClusterIP   10.96.0.1      <none>        443/TCP        20d
kubia                ClusterIP   10.96.170.37   <none>        80/TCP         12h
kubia-nginx          ClusterIP   10.96.154.64   <none>        80/TCP         12h
kubia-svc-nodeport   NodePort    10.96.9.93     <none>        80:32013/TCP   4s
```
分别在2台node的服务器上,查看是否节点服务器是否开启了32013端口

```
#k8s-node1服务器
[work@k8s-node1 ~]$ sudo netstat -tlnp | grep 32013
tcp6       0      0 :::32013                :::*                    LISTEN      30205/kube-proxy
[work@k8s-node1 ~]$

#k8s-node2服务器
[work@k8s-node2 ~]$ sudo netstat -tlnp | grep 32013
tcp6       0      0 :::32013                :::*                    LISTEN      1316/kube-proxy
[work@k8s-node2 ~]$
```

此时在集群外部可以通过访问节点服务器的32013端口访问后端的pods.访问任意一个节点服务器的32013端口,多个节点将流量转发到Service服务.然后Service服务负载到所有的pods,而不仅仅是指定的节点下的pod

```
 ✘ huangyong@huangyong-Macbook-Pro  ~  curl http://172.16.20.252:32013
You've hit kubia-xgnkq
 huangyong@huangyong-Macbook-Pro  ~  curl http://172.16.20.252:32013
You've hit kubia-xgnkq
 huangyong@huangyong-Macbook-Pro  ~  curl http://172.16.20.252:32013
You've hit kubia-rm9qf
 huangyong@huangyong-Macbook-Pro  ~  curl http://172.16.20.252:32013
You've hit kubia-rm9qf
```
可以想象一下,客户端发起的流量到达节点A后,节点A再通过Service转发到另外一台服务器节点B下的pods,这样会造成额外的网络开销和跳转.在Serivce的spec部分定义以下属性的转发策略,可以实现流量转发到接收节点的本地Pods容器

```
spec:
   externalTrafficPolicy: local
   
```
但是这种策略有2个比较大的缺陷:

1. 如果该节点下没有pods可用,流量不会转发到其他可用节点下的pods,而是访问被中断
2. 如果节点A下只有一个PodA,节点B下有2个PodB,则3个节点不会平均负载流量,而是PodA占据50%,PodB下的2个容器各承担25%.


通过节点的真实物理IP访问容错率不高,如果节点服务器出现任何故障,则会中断访问.所以可以在节点前创建一个LoadBalancer类型的负载均衡器.

---

### LoadBalancer

这使得服务可以通过一个专用的负载均衡器来访问，这是由Kubernetes中正在运行的云基础设施提供的。负载均衡器将流量重定向到跨所有节点的节点端口。客户端通过负载均衡器的IP 连接到服务.

使用支持外部负载均衡器的云提供商的服务，设置 type 的值为 "LoadBalancer"，

下面创建一个LoadBalancer的服务类型

```
apiVersion: v1
kind: Service
metadata:
  name: kubia-loadbalancer
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia
    
```
但是由于我们集群不是云厂商提供的.所以无法自动获得负载均衡IP:

```
root@k8s-master ~]# kubectl get svc
NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes           ClusterIP      10.96.0.1      <none>        443/TCP        20d
kubia                ClusterIP      10.96.170.37   <none>        80/TCP         12h
kubia-loadbalancer   LoadBalancer   10.96.1.162    <pending>     80:31924/TCP   10s
```
> 这里我们注意到EXTERNAL-IP一直是pending状态.另外,我们刚才的LoadBlancer配置文件中并没有指定具体的nodePort端口.但是可以看到PORT开放了一个31924的端口.

node节点服务器上也开放了这个端口.

```
[work@k8s-node1 ~]$ sudo netstat -tlnp | grep 31924
tcp6       0      0 :::31924                :::*                    LISTEN      30205/kube-proxy
```

所以在创建NodePort或者LoadBlancer类型的服务时,如果没有指定具体的节点端口,则会随机生成一个节点端口(默认范围:30000-32767)

LoadBalancer类型服务工作在NodePort类型之上.当外部客户端发起请求时,LoadBalancer负载均衡器将流量负载到下面的节点端口,然后节点再转发到k8s的Service,继而再转发到所有的pods容器内部

---

## 总结

* Service用于转发客户端流量到Service标签选择器指定的一组pod容器
* Service可以支持多端口,但是需要使用端口命名
* kubectl get svc可以查看k8s集群中正在运行的Service
* kubectl get endpoints 可以查看服务的IP和端口
* Service默认是ClientIP类型,即通过Service的IP地址访问pod容器,注意该IP不能ping.需要结合IP和port一起使用
* Service还支持NodePort和LoadBalancer两种类型,允许集群外部的客户端访问
* Service的默认FQDN后缀名为svc.cluster.local
* pods容器内部可以通过环境变量来访问Service.Service IP的变量名为:`${ServiceName}_SVC_HOST
Service` port的变量名为: `${ServiceName}_PORT_80_TCP`



