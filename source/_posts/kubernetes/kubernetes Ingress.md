---
title: kuberentes Ingress
date: 2020-06-26 11:59:58
tags: k8s
categories: kubernetes
comments: true
copyright: true
---



## kuberentes Ingress

## 介绍

[K8s ingress 学习博客](https://segmentfault.com/a/1190000019908991)

ingress是暴露服务给外部客户端的另外一种方法.(前面我们学习过2种服务类型可以实现同样的目的:NodePort和LoadBalance)

但是这几种Service都有局限性:

* ClusterIP的默认Serivce方式只能在集群内部访问。
* NodePort:当有几十上百的服务在集群中运行时，NodePort的端口管理是灾难。

* LoadBalance方式受限于云平台，且通常在云平台部署ELB还需要额外的费用。

所幸k8s还提供了一种集群维度暴露服务的方式，也就是ingress。ingress可以简单理解为service的service，他通过独立的ingress对象来制定请求转发的规则，把请求路由到一个或多个service中。这样就把服务与请求规则解耦了，可以从业务维度统一考虑业务的暴露，而不用为每个service单独考虑。

Ingress可以为多个Service提供服务,可以根据请求的HOST和PATH,将请求代理到相关的Service中.而且Ingres工作在HTTP层,可以实现7层代理特性,诸如:coockies和会话亲和力

<!--more-->

---

## ingress与ingress-controller

要理解ingress，需要区分两个概念，ingress和ingress-controller：

* ingress对象：

指的是k8s中的一个api对象，一般用yaml配置。作用是定义请求如何转发到service的规则，可以理解为配置模板。

* ingress-controller：

具体实现反向代理及负载均衡的程序，对ingress定义的规则进行解析，根据配置的规则来实现请求转发。

简单来说，ingress-controller才是负责具体转发的组件，通过各种方式将它暴露在集群入口，外部对集群的请求流量会先到ingress-controller，而ingress对象是用来告诉ingress-controller该如何转发请求，比如哪些域名哪些path要转发到哪些服务等等。

---

## ingress-controller

ingress-controller并不是k8s自带的组件，实际上ingress-controller只是一个统称，用户可以选择不同的ingress-controller实.

目前，由k8s维护的ingress-controller只有google云的GCE与ingress-nginx两个，其他还有很多第三方维护的ingress-controller，具体可以参考官方文档。

但是不管哪一种ingress-controller，实现的机制都大同小异，只是在具体配置上有差异。一般来说，ingress-controller的形式都是一个pod，里面跑着daemon程序和反向代理程序。daemon负责不断监控集群的变化，根据ingress对象生成配置并应用新配置到反向代理，比如nginx-ingress就是动态生成nginx配置，动态更新upstream，并在需要的时候reload程序应用新配置。

为了方便，后面的例子都以k8s官方维护的nginx-ingress为例。

---

## ingress

ingress是一个API对象，和其他对象一样，通过yaml文件来配置。ingress通过http或https暴露集群内部service，给service提供外部URL、负载均衡、SSL/TLS能力以及基于host的方向代理。ingress要依靠ingress-controller来具体实现以上功能。

---

## ingress的部署

ingress的部署，需要考虑两个方面：

1.ingress-controller是作为pod来运行的，以什么方式部署比较好

2.ingress解决了把如何请求路由到集群内部，那它自己怎么暴露给外部比较好


下面列举一些目前常见的部署和暴露方式，具体使用哪种方式还是得根据实际需求来考虑决定。

**Deployment+LoadBalancer模式的Service**

如果要把ingress部署在公有云，那用这种方式比较合适。用Deployment部署ingress-controller，创建一个type为LoadBalancer的service关联这组pod。大部分公有云，都会为LoadBalancer的service自动创建一个负载均衡器，通常还绑定了公网地址。只要把域名解析指向该地址，就实现了集群服务的对外暴露。

**Deployment+NodePort模式的Service**

同样用deployment模式部署ingress-controller，并创建对应的服务，但是type为NodePort。这样，ingress就会暴露在集群节点ip的特定端口上。由于nodeport暴露的端口是随机端口，一般会在前面再搭建一套负载均衡器来转发请求。该方式一般用于宿主机是相对固定的环境ip地址不变的场景。
NodePort方式暴露ingress虽然简单方便，但是NodePort多了一层NAT，在请求量级很大时可能对性能会有一定影响。

**DaemonSet+HostNetwork+nodeSelector**

用DaemonSet结合nodeselector来部署ingress-controller到特定的node上，然后使用HostNetwork直接把该pod与宿主机node的网络打通，直接使用宿主机的80/433端口就能访问服务。这时，ingress-controller所在的node机器就很类似传统架构的边缘节点，比如机房入口的nginx服务器。该方式整个请求链路最简单，性能相对NodePort模式更好。缺点是由于直接利用宿主机节点的网络和端口，一个node只能部署一个ingress-controller pod。比较适合大并发的生产环境使用。

---

## ingress-controller部署

参考文档: [kubernetes集群部署](https://www.kuboard.cn/install/install-k8s.html#%E5%AE%89%E8%A3%85-ingress-controller)

---

## 配置一个简单的Ingress对象

编辑一个简单的yaml配置文件如下:

```
[root@k8s-master ~]# cat kubia-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
    - host: kubia.example.com 
      http:
        paths:
        - path: /
          backend:
            #将访问请求转发到上一节中创建的kubia-svc-nodeport的Serivce上
            serviceName: kubia-svc-nodeport
            #kubia-svc-nodeport的源端口
            servicePort: 80
            
```

接下来,可以将kubia.exanple.com域名解析到任意一个Node服务器节点的IP地址,然后请求这个域名

```
 huangyong@huangyong-Macbook-Pro  ~  curl http://kubia.example.com
You've hit kubia-hd429
 huangyong@huangyong-Macbook-Pro  ~  curl http://kubia.example.com
You've hit kubia-fcc59
 huangyong@huangyong-Macbook-Pro  ~  curl http://kubia.example.com
You've hit kubia-rm9qf
```
----

## ingress关联多个Service

ingress的rules字段可以包含多个转发规则,可以将不同的访问URL或者HOST代理到相关联的不同的Service中.参考下面的例子

* 不同的PATH转发到不同的Service

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia-ingress-kubia

spec:
  rules:
    - host: kubia.example.com
      paths:
       - path: /foo
         backend:
            serviceName: kubia-svc-foo
            servicePort: 80

       - path: /bar
         backend:
            serviceName: kubia-svc-bar
            servicePort: 80
```

* 不同的Host转发到不同的Service

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia-ingress-kubia2

spec:
  rules:
    - host: foo.example.com
      paths:
       - path: /
         backend:
            serviceName: kubia-svc-foo
            servicePort: 80
    - host: bar.example.com
      paths:
       - path: /
         backend:
            serviceName: kubia-svc-bar
            servicePort: 80
            
```
---

## Ingress处理https TLS传输

当用户使用TLS链接时,ingress负责处理用户的TLS请求,但是和后端的Pod仍然是以http连接,要实现这一点,Https的证书和秘钥必须挂载在ingress控制器上.

证书和秘钥可以保存在kubernetes的secret资源对象中.然后在ingress的mainfest资源中引用.

下面先创建一个自签名的证书和私钥

```
[root@k8s-master ~]# openssl genrsa -out tls.key 2048

[root@k8s-master ~]#openssl req -new -x509 -key tls.key -out tls.cert -days 360 -subj /CN=kubia.example.com

```

创建Secret

```
[root@k8s-master ~]# kubectl create secret tls tls-secret --cert=tls.cert --key=tls.key
secret/tls-secret created
```

### 创建https的ingress

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  tls:
   - hosts:
     - kubia.example.com
     secretName: tls-secret

  rules:
   - host: kubia.example.com
     http:
       paths:
         - path: /
           backend:
              serviceName: kubia
              servicePort: 80

```

更新ingress资源kubia

```
[root@k8s-master ~]# kubectl apply -f kubia-ingress-tls.yaml
ingress.extensions/kubia configured
```

ingress支持443端口

```
[root@k8s-master ~]# kubectl get ing
NAME    HOSTS               ADDRESS   PORTS     AGE
kubia   kubia.example.com             80, 443   6h4m
```
现在客户端可以https访问kubia.expample.com

```
 ✘ huangyong@huangyong-Macbook-Pro  ~  curl -k https://kubia.example.com
You've hit kubia-xgnkq
```






