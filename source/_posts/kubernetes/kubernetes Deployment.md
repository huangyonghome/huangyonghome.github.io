---
title: kuberentes Deployment
date: 2020-06-26 11:59:58
tags: k8s
categories: kubernetes
comments: true
copyright: true
---



## kuberentes Deployment

## 介绍

之前学习过使用ReplicationController和ReplicaSet实现托管和更新pod容器.
客户端通过Service服务访问RC或者RS托管的一组pod.这也是Kubernetes典型的应用程序运行方式

假设这个时候应用程序需要更新,从v1版本更新到v2,有以下两种方式更新pod镜像:

* 直接删除所有现有的pod,然后创建新的pod
* 可以先创建新的Pod,并等待他们运行成功后,再删除旧的.

这2中方法各有优缺点.第一种方法将导致服务在端在时间内不可用.第二种方法要求应用程序同时支持2个版本对外提供服务.

本文具体介绍kubernetes上述两种更新方式.然后介绍通过手动方式,replicatController的更新方式.最后再引入deployment机制的用法介绍

<!--more-->

----

## 1.更新pod

### 删除旧版本pod,使用新版本pod替换

这是更新pod最简单的方式,如果有一个replicationController管理一组v1的pod,可以直接修改pod模板为v2版本.然后执行下列命令删除旧rc,创建新rc来更新整个pod组

```
[root@k8s-master ~]# kubectl delete -f kubia-rc.yaml
[root@k8s-master ~]# kubectl create -f kubia-rc.yaml
replicationcontroller/kubia created

[root@k8s-master ~]# kubectl get pods
NAME                READY   STATUS        RESTARTS   AGE
kubia-2xz92         0/1     Terminating   0          92s
kubia-dkhsz         1/1     Running       0          3s
kubia-kn694         1/1     Running       0          3s
kubia-l946z         1/1     Running       0          3s
kubia-p5bmx         0/1     Terminating   0          92s
kubia-sjxlz         0/1     Terminating   0          92s
kubia-tt6fz         0/1     Terminating   0          92s
kubia-xm8js         1/1     Running       0          3s
ssd-monitor-h7k7q   1/1     Running       0          57s
```
此时旧版本会处于terminating终止中状态,(最后会逐渐被删除),同时创建了新的一组v2版本的Pod.

---

### 先新建pod,再删除旧pod

如果短暂的服务中断不可介绍,并且应用程序支持多个版本同时对外服务.那么可以先创建新的Pod,然后删除原有的pod.

> 但是这会在短暂时间内同时运行2倍数量的pod,要求服务器有更多的空闲硬件资源

#### 大致步骤

1. 创建新版本pod
2. 一旦新版本Pod就绪,将Service服务从老版本pod关联到新版本pod
3. 删除旧的ReplicationController

---

## 2.使用ReplicationController实现自动的滚动升级

### 使用kubectl命令手动滚动式升级


#### 1.下面创建一个ReplicationController和一个v1版本进行的pod.以及一个nodePort类型的Service.并以此为例子进行后面的演练

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia-dm-rc-v1
spec:
  replicas: 3
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
       - image: luksa/kubia:v1
         name: nodejs

---
#一个yaml文件可以定义多种资源,中间用---隔开

apiVersion: v1
kind: Service
metadata:
  name: kubia-dm-svc-v1

spec:
  type: NodePort
  selector:
    app: kubia

  ports:
  - port: 80
    targetPort: 8080
    nodePort: 32013

```

这个例子中创建了一个kubia-dm-rc-v1的ReplicationController.并运行了3个副本.另外还创建了一个kubia-dm-svc-v1的nodeport类型的Service

```
[root@k8s-master ~]# kubectl create -f deployment-kubia-v1.yaml
replicationcontroller/kubia-dm-rc-v1 created
service/kubia-dm-svc-v1 created

[root@k8s-master ~]# kubectl get svc
NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
fortune-nfs-svc      ClusterIP      10.96.243.6     <none>        80/TCP         14d
fortune-svc          ClusterIP      10.96.123.110   <none>        80/TCP         14d
kubernetes           ClusterIP      10.96.0.1       <none>        443/TCP        41d
kubia                ClusterIP      10.96.170.37    <none>        80/TCP         21d
kubia-dm-svc-v1      NodePort       10.96.136.55    <none>        80:32013/TCP   4s
kubia-loadbalancer   LoadBalancer   10.96.1.162     <pending>     80:31924/TCP   20d
kubia-nginx          ClusterIP      10.96.154.64    <none>        80/TCP         21d

[root@k8s-master ~]# kubectl get pods
NAME                   READY   STATUS    RESTARTS   AGE
kubia-dm-rc-v1-dsk5v   1/1     Running   0          9s
kubia-dm-rc-v1-mk6hf   1/1     Running   0          9s
kubia-dm-rc-v1-rh6qz   1/1     Running   0          9s
[root@k8s-master ~]#

```

访问Service的IP:10.96.136.55,或者节点物理网卡IP的32013端口访问pod容器,目前是v1版本

```
[root@k8s-master ~]# while true;do curl http://10.96.136.55;sleep 1;done
This is v1 running in pod kubia-dm-rc-v1-dsk5v
This is v1 running in pod kubia-dm-rc-v1-mk6hf
This is v1 running in pod kubia-dm-rc-v1-rh6qz
This is v1 running in pod kubia-dm-rc-v1-rh6qz
This is v1 running in pod kubia-dm-rc-v1-rh6qz
This is v1 running in pod kubia-dm-rc-v1-mk6hf
```

#### 2.创建v2版本的应用.这里直接使用luksa/kubia:v2

> 虽然在开发的过程中,进程推送修改后的应用到同一个镜像tag,但是这种做法不推荐.如果用的是latest的tag倒还好,因为不指定容器tag,或者tag是latest,则镜像拉取策略(imagePullPolicy)默认为always.

>如果使用了指定的tag,例如(v1),则默认的镜像拉取策略为IfNotPresent,则虽然应用程序更新了,但是由于tag不变(都是v1),所以某些已经拉取了v1版本的镜像的Pod仍然使用v1旧版本的镜像.但是之前没有运行该pod的节点拉取的是v2版本.这就造成虽然镜像名一样,但是内容不一样的意外情况

#### 3.使用`kubectl rolling-update`命令开始ReplicationController的滚动升级.

>在这之前打开一个新终端,循环访问Pod容器,观察升级过程中客户端访问情况

命令格式为

```
kubectl rolling-update 旧rc名 新rc名 --image=新镜像
```

```
[root@k8s-master ~]# kubectl rolling-update kubia-dm-rc-v1 kubia-dm-rc-v2 --image=luksa/kubia:v2
Command "rolling-update" is deprecated, use "rollout" instead
Created kubia-dm-rc-v2

Scaling up kubia-dm-rc-v2 from 0 to 3, scaling down kubia-dm-rc-v1 from 3 to 0 (keep 3 pods available, don't exceed 4 pods)
Scaling kubia-dm-rc-v2 up to 1
Scaling kubia-dm-rc-v1 down to 2
Scaling kubia-dm-rc-v2 up to 2
```

该命令会自动进行滚动升级,先创建个v2版本(初始副本为0),然后v2版本副本慢慢增加到3,同时v1版本缩容慢慢缩减到0

此时访问pod容器的nginx服务并没有中断,访问流量逐渐慢慢从v1过度到v2

```
This is v1 running in pod kubia-dm-rc-v1-rh6qz
This is v1 running in pod kubia-dm-rc-v1-dsk5v
This is v1 running in pod kubia-dm-rc-v1-dsk5v
This is v2 running in pod kubia-dm-rc-v2-nvjln
This is v2 running in pod kubia-dm-rc-v2-nvjln
This is v1 running in pod kubia-dm-rc-v1-dsk5v
This is v1 running in pod kubia-dm-rc-v1-dsk5v
This is v2 running in pod kubia-dm-rc-v2-nvjln
This is v1 running in pod kubia-dm-rc-v1-rh6qz
This is v2 running in pod kubia-dm-rc-v2-nvjln
This is v1 running in pod kubia-dm-rc-v1-dsk5v
This is v2 running in pod kubia-dm-rc-v2-nvjln
This is v2 running in pod kubia-dm-rc-v2-nvjln
This is v1 running in pod kubia-dm-rc-v1-rh6qz
This is v2 running in pod kubia-dm-rc-v2-gx2qr
This is v1 running in pod kubia-dm-rc-v1-dsk5v
This is v1 running in pod kubia-dm-rc-v1-dsk5v
This is v1 running in pod kubia-dm-rc-v1-dsk5v
```

查看kubia-dm-rc-v2新的rc,发现多了一个label标签

```
[root@k8s-master ~]# kubectl describe rc kubia-dm-rc-v2
.....
  Labels:  app=kubia
           deployment=e765c85861762e0e8a6a68870e355478
.....
```

同时旧的kubia-dm-rc-v1的rc,也多了一个label标签,标签末尾有个orig

```
[root@k8s-master ~]# kubectl describe rc kubia-dm-rc-v1
......
  Labels:  app=kubia
           deployment=0a4a2fdf1a43e220276c99beda132b0c-orig
......
```

甚至正在运行的pod也修改了标签,3个正在运行的老的Pod容器副本增加了对应的旧的rc标签.

同时创建了个新的pod,标签对应为新的kubia-dm-rc-v2新rc的标签

```
[root@k8s-master ~]# kubectl get po --show-labels
NAME                   READY   STATUS    RESTARTS   AGE   LABELS
kubia-dm-rc-v1-dsk5v   1/1     Running   0          18m   app=kubia,deployment=0a4a2fdf1a43e220276c99beda132b0c-orig
kubia-dm-rc-v1-mk6hf   1/1     Running   0          18m   app=kubia,deployment=0a4a2fdf1a43e220276c99beda132b0c-orig
kubia-dm-rc-v1-rh6qz   1/1     Running   0          18m   app=kubia,deployment=0a4a2fdf1a43e220276c99beda132b0c-orig
kubia-dm-rc-v2-nvjln   1/1     Running   0          59s   app=kubia,deployment=e765c85861762e0e8a6a68870e355478

```

最终,老的ReplicationController副本会缩容到0,导致最后一个v1 pod被删除,老的kubia-dm-rc-v1名字rc被删除.

新的ReplicationController副本扩容到3,最后所有的请求都到了v2

```
[root@k8s-master ~]# kubectl get rc
NAME             DESIRED   CURRENT   READY   AGE
kubia-dm-rc-v2   3         3         3       15m
[root@k8s-master ~]#

[root@k8s-master ~]# while true;do curl http://10.96.136.55;sleep 1;done
This is v2 running in pod kubia-dm-rc-v2-gx2qr
This is v2 running in pod kubia-dm-rc-v2-gx2qr
This is v2 running in pod kubia-dm-rc-v2-nvjln
This is v2 running in pod kubia-dm-rc-v2-l44qp
This is v2 running in pod kubia-dm-rc-v2-gx2qr
^C
```
---

## 3.使用Deployment声明式升级应用

使用kubectl命令并不是最优的方案,一方面这是由kubectl客户端执行任务,并不是kubernetes master执行的.领一方面,如果在kubectl执行升级时,出现网络故障,升级过程将会终端.造成意料之外情况

最重要的是,通过命令行来手动的升级并不符合Kubernetes的理念和预期.直接使用期望的副本数来伸缩pod是kubernetes期望的部署和伸方式.

正是这一点推动了一种称为Deployment的新资源的引入.也是现在kubernetes部署应用程序的首选方式

----

### 3.1 Deployment介绍

Deployment是一种更高阶的资源,当创建一个Deployment时,ReplicaSet资源也会随之创建.ReplicaSet是新一代的ReplicationController,并且已经替代了后者来管理pod.

在使用Deployment时,并不是由Deployment直接创建和管理pod.而是Deployment创建ReplicaSet,然后ReplicaSet创建pod

```
Deployment-----> ReplicaSet----->pods1,pods2,pods3...... 
```

> 为什么这么复杂,引入Deployment概念,直接ReplicaSet管理pod不是已经挺好的吗?.在上个滚动升级的例子中,我们需要协调2个ReplicationController.新的版本扩容,旧的版本缩容,所以这需要另一个资源来协调.

> Deployment资源就是负责处理这个问题

#### 1.创建一个Deployment

创建deployment和创建rc,rs并没有任何区别,也是由标签选择器,期望副本,和pod模板组成.另外还包含另一个字段,指定一个部署策略

稍微修改上一个rc的配置

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubia-dm-v1
spec:
  replicas: 3
  selector:
      matchLabels:
         app: kubia
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
       - image: luksa/kubia:v1
         name: nodejs

---
#一个yaml文件可以定义多种资源,中间用---隔开

apiVersion: v1
kind: Service
metadata:
  name: kubia-dm-svc-v1

spec:
  type: NodePort
  selector:
    app: kubia

  ports:
  - port: 80
    targetPort: 8080
    nodePort: 32014
    
```

deployment属于apps/v1版本.大体字段差不多,但是selector字段必须要显示指定.

#### 2,创建depolyment和Service

```
[root@k8s-master ~]# kubectl create -f deployment-kubia-v2.yaml --record
deployment.apps/kubia-dm-v1 created
service/kubia-dm-svc-v1 created
[root@k8s-master ~]#
```

> --record选项会记录历史版本号

#### 3.展示Deployment滚动过程中的状态

通过以下命令可以看到deployment部署的pod已经全部正常就位

```
[root@k8s-master ~]# kubectl rollout status deployment kubia-dm-v1
deployment "kubia-dm-v1" successfully rolled out

[root@k8s-master ~]# kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
kubia-dm-v1-74fb644f8-6fp7n   1/1     Running   0          2m32s
kubia-dm-v1-74fb644f8-8hddk   1/1     Running   0          2m32s
kubia-dm-v1-74fb644f8-hcrck   1/1     Running   0          2m32s

```

通过deploy创建的pod命名和之前的有些稍微区别.中间的一串数字是deployment和replicaSet模板的哈希值.查看rs的信息,可以看到rs名称中也包含了pod相同的哈希值

```
[root@k8s-master ~]# kubectl get replicasets
NAME                    DESIRED   CURRENT   READY   AGE
kubia-dm-v1-74fb644f8   3         3         3       3m58s
```

### 3.2.deployment升级

更新升级deployment相比之前的一个例子要简单,并且强大的多.只需要修改deployment资源定义模板配置文件中的镜像tag和期望副本.kubernetes会自动扩容或者缩容,以匹配期望的状态

##### 不同的deployment升级策略

deployment有2种升级策略

* RollingUpdate(默认策略): 滚动更新.先删除一个旧的Pod,然后创建一个新的Pod,这就是蓝绿发布
* Recreate:先全部删除旧pod,然后创建新Pod.

其中RollingUpdate还有更详细的配置参数定义滚动过程动作.

> 在演示滚动更新之前,先回顾一下修改对象的几种不同方式.之前接触过通过`kubectl edit`命令修改,或者通过`kubectl patch`打补丁的方式修改

下表中是几种不同的修改方式:


方法  | 例子| 作用
---|---|---
kubectl edit | kubectl edit deployment kubia-dm-v2| 使用默认编辑器打开资源配置修改并更新
kubectl patch | kubectl patch deployment kubia-dm-v2 -p '{"spec":{"replicas":4}}' | 通过打补丁的方式修改单个资源属性,比如修改副本数为4
kubectl apply| kubectl apply -f kubia-dm-v2 | 应用一个完整的yaml文件,如果yaml中指定的对象不存在,则会被创建,文件中新的值会更新.
kubectl replace|kubectl replace -f kubia-dm-v2 | 也是应用一个新的完整的yaml文件,文件中定义的值会更新原有的值,但是与apply相反,要求新的对象必须事先存在,否则会报错.
kubectl set image| kubectl set iamge deployment kubia-dm-v2 nodejs=luksa/kubia:v2 | 修改部分资源(pod,job,rc,rs,deployment,DemonSet)的镜像

----

接下来演示滚动升级.首先通过patch方式修改deployment的一个属性

```
[root@k8s-master ~]# kubectl patch deployment kubia-dm-v1 -p '{"spec":{"minReadySeconds":10}}'
deployment.apps/kubia-dm-v1 patched
[root@k8s-master ~]#

```

在另一个终端持续访问pod的nginx服务

```
[root@k8s-master ~]# while true;do curl http://10.96.68.186;sleep 1;done
```

接下来通过另外一种方式修改deployment中定义的pod镜像版本为v2

```
[root@k8s-master ~]# kubectl set image deployment kubia-dm-v1 nodejs=luksa/kubia:v2
deployment.apps/kubia-dm-v1 image updated
[root@k8s-master ~]#
```
此时,kuberntes开始执行滚动更新,直到所有的v1pod都被删除,所有的请求到v2

```
This is v1 running in pod kubia-dm-v1-74fb644f8-6fp7n
This is v1 running in pod kubia-dm-v1-74fb644f8-6fp7n
This is v1 running in pod kubia-dm-v1-74fb644f8-6fp7n
This is v1 running in pod kubia-dm-v1-74fb644f8-6fp7n
This is v2 running in pod kubia-dm-v1-6c99f46f5-fj4dl
This is v1 running in pod kubia-dm-v1-74fb644f8-6fp7n
This is v2 running in pod kubia-dm-v1-6c99f46f5-fj4dl
This is v2 running in pod kubia-dm-v1-6c99f46f5-b55rf
This is v2 running in pod kubia-dm-v1-6c99f46f5-b55rf
This is v2 running in pod kubia-dm-v1-6c99f46f5-b55rf
This is v2 running in pod kubia-dm-v1-6c99f46f5-fj4dl
This is v2 running in pod kubia-dm-v1-6c99f46f5-b55rf
```

deploy进行滚动更新的过程和上个例子中使用的`kubectl rolling-update`命令非常相似.稍有不同的是,deployment更新完成后,创建了一个新的ReplicaSet.但是旧的ReplicaSet并不会删除.

```
[root@k8s-master ~]# kubectl get rs
NAME                    DESIRED   CURRENT   READY   AGE
kubia-dm-v1-6c99f46f5   3         3         3       5m1s
kubia-dm-v1-74fb644f8   0         0         0       61m
```

但是相比上个例子,使用deployment进行滚动升级明显要方便的多,用户仅仅需要管理一个单一的deployment资源.并不需要过多的关心pod和RS.

---

### 3.3 deployment回滚

##### 1.使用v3版本的镜像(luksa/kubia:v3)升级,但是v3版本在处理第五个请求时会返回500错误,导致服务器不可用.

##### 2.修改deployment资源kubia-dm-v1的镜像为v3版本,出发滚动升级

```
[root@k8s-master ~]# kubectl set image deployment kubia-dm-v1 nodejs=luksa/kubia:v3
deployment.apps/kubia-dm-v1 image updated
```

滚动升级完成后,全部访问到v3版本镜像,但是访问出现问题

```
This is v2 running in pod kubia-dm-v1-6c99f46f5-fj4dl
This is v2 running in pod kubia-dm-v1-6c99f46f5-fj4dl
^@This is v2 running in pod kubia-dm-v1-6c99f46f5-fj4dl
Some internal error has occurred! This is pod kubia-dm-v1-6bb646bc8d-q7lr4
Some internal error has occurred! This is pod kubia-dm-v1-6bb646bc8d-q7lr4
This is v3 running in pod kubia-dm-v1-6bb646bc8d-p9nq5
Some internal error has occurred! This is pod kubia-dm-v1-6bb646bc8d-q7lr4
Some internal error has occurred! This is pod kubia-dm-v1-6bb646bc8d-qzwss
This is v3 running in pod kubia-dm-v1-6bb646bc8d-p9nq5
This is v2 running in pod kubia-dm-v1-6c99f46f5-fj4dl
This is v3 running in pod kubia-dm-v1-6bb646bc8d-p9nq5
Some internal error has occurred! This is pod kubia-dm-v1-6bb646bc8d-q7lr4
Some internal error has occurred! This is pod kubia-dm-v1-6bb646bc8d-q7lr4
Some internal error has occurred! This is pod kubia-dm-v1-6bb646bc8d-q7lr4
Some internal error has occurred! This is pod kubia-dm-v1-6bb646bc8d-q7lr4
This is v3 running in pod kubia-dm-v1-6bb646bc8d-p9nq5
Some internal error has occurred! This is pod kubia-dm-v1-6bb646bc8d-p9nq5
Some internal error has occurred! This is pod kubia-dm-v1-6bb646bc8d-q7lr4
Some internal error has occurred! This is pod kubia-dm-v1-6bb646bc8d-qzwss
Some internal error has occurred! This is pod kubia-dm-v1-6bb646bc8d-qzwss
Some internal error has occurred! This is pod kubia-dm-v1-6bb646bc8d-q7lr4
```

##### 3.手动停止回滚升级

先演示如何手动停止升级.deployment可以非常容易的回滚到先前部署的版本.它可以让Kubernetes取消最后一次部署的Deployment

命令: `kubectl rollout undo deployment DeplymentName`

```
[root@k8s-master ~]# kubectl rollout undo deployment kubia-dm-v1
deployment.apps/kubia-dm-v1 rolled back
```

此时deployment会回滚到上一个版本

```
Some internal error has occurred! This is pod kubia-dm-v1-6bb646bc8d-qzwss
Some internal error has occurred! This is pod kubia-dm-v1-6bb646bc8d-q7lr4
This is v2 running in pod kubia-dm-v1-6c99f46f5-pkv5f
This is v2 running in pod kubia-dm-v1-6c99f46f5-vnglb
Some internal error has occurred! This is pod kubia-dm-v1-6bb646bc8d-q7lr4
This is v2 running in pod kubia-dm-v1-6c99f46f5-vnglb
Some internal error has occurred! This is pod kubia-dm-v1-6bb646bc8d-q7lr4
Some internal error has occurred! This is pod kubia-dm-v1-6bb646bc8d-qzwss
This is v2 running in pod kubia-dm-v1-6c99f46f5-pkv5f
```

##### 4. 显示deployment的滚动升级历史

命令: `kubectl rollout history deployment DeploymentName`

```
[root@k8s-master ~]# kubectl rollout history deployment kubia-dm-v1
deployment.apps/kubia-dm-v1
REVISION  CHANGE-CAUSE
1         kubectl create --filename=deployment-kubia-v2.yaml --record=true
3         kubectl create --filename=deployment-kubia-v2.yaml --record=true
4         kubectl create --filename=deployment-kubia-v2.yaml --record=true
```

使用--revison=REVISION编号,可以看回滚历史的具体信息,比如

```
[root@k8s-master ~]# kubectl rollout history  deployment kubia-dm-v1 --revision=3
deployment.apps/kubia-dm-v1 with revision #3
Pod Template:
  Labels:	app=kubia
	pod-template-hash=6bb646bc8d
  Annotations:	kubernetes.io/change-cause: kubectl create --filename=deployment-kubia-v2.yaml --record=true
  Containers:
   nodejs:
    Image:	luksa/kubia:v3
    Port:	<none>
    Host Port:	<none>
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
```

> 还记得创建deployment时添加的--record参数吗?

##### 5. 回滚到一个特定的deployment版本

有了上面的revision版本编号,可以回滚到一个特定的deployment版本,比如回滚到第一个版本

命令: `kubectl rollout undo deployment Deployment名 --to-revision=回滚版本`

```
[root@k8s-master ~]# kubectl rollout undo deployment kubia-dm-v1 --to-revision=1
deployment.apps/kubia-dm-v1 rolled back
```
现在全部是v1版本

```
This is v1 running in pod kubia-dm-v1-74fb644f8-fckvz
This is v1 running in pod kubia-dm-v1-74fb644f8-fckvz
This is v1 running in pod kubia-dm-v1-74fb644f8-lf4fv
This is v1 running in pod kubia-dm-v1-74fb644f8-lf4fv
This is v1 running in pod kubia-dm-v1-74fb644f8-fckvz
This is v1 running in pod kubia-dm-v1-74fb644f8-lf4fv
This is v1 running in pod kubia-dm-v1-74fb644f8-lf4fv
This is v1 running in pod kubia-dm-v1-74fb644f8-lf4fv
```

再次查看回滚历史,发现revision版本4使用的是v2版本..所以再试一下回滚到v2版本也就是revision=4

```
[root@k8s-master ~]# kubectl rollout history  deployment kubia-dm-v1 --revision=4
deployment.apps/kubia-dm-v1 with revision #4
Pod Template:
  Labels:	app=kubia
	pod-template-hash=6c99f46f5
  Annotations:	kubernetes.io/change-cause: kubectl create --filename=deployment-kubia-v2.yaml --record=true
  Containers:
   nodejs:
    Image:	luksa/kubia:v2
    Port:	<none>
    Host Port:	<none>
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
```

 回滚到revision=4

 ```
 [root@k8s-master ~]# kubectl rollout undo deployment kubia-dm-v1 --to-revision=4
deployment.apps/kubia-dm-v1 rolled back
 ```

于是,开始回滚到v2版本镜像

```
This is v1 running in pod kubia-dm-v1-74fb644f8-fckvz
This is v1 running in pod kubia-dm-v1-74fb644f8-lf4fv
This is v1 running in pod kubia-dm-v1-74fb644f8-lf4fv
This is v1 running in pod kubia-dm-v1-74fb644f8-fckvz
This is v2 running in pod kubia-dm-v1-6c99f46f5-458p7
This is v1 running in pod kubia-dm-v1-74fb644f8-fckvz
This is v1 running in pod kubia-dm-v1-74fb644f8-bmwx7
This is v1 running in pod kubia-dm-v1-74fb644f8-lf4fv
This is v1 running in pod kubia-dm-v1-74fb644f8-fckvz
This is v1 running in pod kubia-dm-v1-74fb644f8-bmwx7
This is v2 running in pod kubia-dm-v1-6c99f46f5-458p7
This is v2 running in pod kubia-dm-v1-6c99f46f5-cvbfd
```

##### 6. 历史ReplicaSet版本记录

查看rs信息,可以发现保留了3个RS版本,一份是正在使用的(v2),一份是上一次回滚的版本(v1),还有一份是最早的版本(v3))

```
[root@k8s-master ~]# kubectl get rs
NAME                     DESIRED   CURRENT   READY   AGE
kubia-dm-v1-6bb646bc8d   0         0         0       22m
kubia-dm-v1-6c99f46f5    3         3         3       20h
kubia-dm-v1-74fb644f8    0         0         0       20h

```
通过回滚历史中的pod-template-hash的号码,可以定位到某个rs名字对应的Pod镜像版本以及rs对应的回滚版本

从下面的命令中可以看到6bb646bc8d的HASH值对应的是revision=3这个版本

```
[root@k8s-master ~]# kubectl rollout history deployment kubia-dm-v1 --revision=3
deployment.apps/kubia-dm-v1 with revision #3
Pod Template:
  Labels:	app=kubia
	pod-template-hash=6bb646bc8d
  Annotations:	kubernetes.io/change-cause: kubectl create --filename=deployment-kubia-v2.yaml --record=true
  Containers:
   nodejs:
    Image:	luksa/kubia:v3
    Port:	<none>
    Host Port:	<none>
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
```

 > 旧版本过多会导致ReplicaSet列表管理混乱,可以通过Deployment的revisionHistoryLimit属性来限制历史版本数量,默认是3

---

### 4 控制滚动升级速率

在上面的例子中,进行deployment升级时会看到第一个新pod被创建(等待就绪后),然后旧pod被删除,然后又一个新pod被创建,然后又一个旧pod被删除,直到新pod创建完毕,旧pod全部删除完

在此期间有2个滚动策略的属性决定一次性替换多少个Pod.这就是maxSurge字段和maxUnavailable字段.

这2个字段的属性和用法可以通过下面的命令查看

```
kubectl explain deployment.spec.strategy.rollingUpdate
```
* maxSurge: 该字段决定了deployment期望的副本之外,最多运行超出的Pod实例数量.默认为25%.也就是pod实例最多可以比期望数量多25%.也可以是一个绝对值.如果期望副本数是4.那么不会同时运行超过5个pod.

* maxUnavailable: 该字段决定了在滚动升级期间,**相对于期望副本数**有多少个Pod实例可以处于不可用状态.默认值也是25%.所以可用的Pod实例数不能低于期望副本数的75%.也可以是一个绝对值.如果副本数是4,那么最多只允许有一个Pod处于不可用状态.

> 需要注意的是maxUnavailable是相对期望副本数而言的.并不是绝对值.例如期望副本数是3,maxSurge和maxUnavailable都设置成1.那么第一步,先删除1个旧Pod,然后创建2个新pod,此时一共4个Pod,但是由于新Pod还未就绪,所以一共只有2个旧pod提供服务..虽然maxUnavailable是1.但是有2个pod不可用.但是由于期望副本数是3.所以相对副本数来说,仍然是只有1个pod不可用


### 5.滚动升级暂停(金丝雀发布)

上一个v3版本出现了Bug,如果采用这种升级方式,势必会带来糟糕的体验.比较好的方式是采用金丝雀发布,在现有的版本之上运行一个新版本pod.并查看一小部分用户请求的情况.如果一旦没问题.就可以用新的pod替换所有的旧的pod

deploy可以使用暂停滚动升级来实现这一需求,在触发滚动升级刚开始几秒后,马上暂停滚动更新.下面用v4版本的镜像来演示这个例子.


```
[root@k8s-master ~]# kubectl set image deployment kubia-dm-v1  nodejs=luksa/kubia:v4
deployment.apps/kubia-dm-v1 image updated
```
使用命令`kubectl rollout pause deployment deploymentName`暂停

```
[root@k8s-master ~]# kubectl rollout pause deployment kubia-dm-v1
deployment.apps/kubia-dm-v1 paused
```

此时增加了一个新版本的Pod,只有一个v4 pod正在运行,

```
[root@k8s-master ~]# kubectl get pods
NAME                           READY   STATUS    RESTARTS   AGE
kubia-dm-v1-6c99f46f5-458p7    1/1     Running   0          4h43m
kubia-dm-v1-6c99f46f5-cvbfd    1/1     Running   0          4h43m
kubia-dm-v1-6c99f46f5-q4lr6    1/1     Running   0          4h42m
kubia-dm-v1-8585b844df-5rnx7   1/1     Running   0          93s
```
```
This is v2 running in pod kubia-dm-v1-6c99f46f5-458p7
This is v2 running in pod kubia-dm-v1-6c99f46f5-cvbfd
This is v2 running in pod kubia-dm-v1-6c99f46f5-cvbfd
This is v4 running in pod kubia-dm-v1-8585b844df-5rnx7
This is v2 running in pod kubia-dm-v1-6c99f46f5-cvbfd
```

#### 恢复滚动升级

一旦确认v4版本pod没有问题,就可以恢复升级了,使用命令`kubectl rollout resume deployment deploymentName`

```
[root@k8s-master ~]# kubectl rollout resume deployment kubia-dm-v1
deployment.apps/kubia-dm-v1 resumed
```

> 在滚动升级的过程中,想要在一个确切的位置暂停滚动升级,目前还无法做到,目前想要实现金丝雀发布的正确方式是使用2个不同的deployment,并同时调整他们对应的pod数量

> 暂停滚动升级的另一个用处是,先暂停滚动升级,然后可以反复多次修改deployment配置文件,从而不会触发自动升级

### 6. minReadySeconds用处

之前演示过使用minReadySeconds参数.这个属性是指新创建的pod至少要成功运行多久后,才能视为可用状态,从而继续删除旧Pod,新建新pod.配合就绪探针的使用,当达到这个时间,且pod就绪探针正常时,pod才被标记就绪状态.如果探针返回失败,新的Pod运行出错,那么新版本的滚动升级将被阻止.

使用这个属性可以减缓滚动升级的过程,同时能有效的组织升级继续进行.在生产中,需要设置为较高的值.以确保pod在开始接收流量后持续保持就绪状态

----

### 7.配合就绪探针阻止有Bug版本的滚动部署

再次部署v3版本,但是这次配合就绪探针和minReadySeconds参数来演示阻止滚动升级.

下面定义deployment-kubia-with-readinesscheck.yaml文件来更新deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubia-dm-v1
spec:
  replicas: 3
  #Pod就绪等待时间
  minReadySeconds: 10
  #滚动更新策略
  strategy:
    rollingUpdate:
       maxSurge: 1
       maxUnavailable: 0 #设置为0.确保pod在升级过程中被挨个替代
    type: RollingUpdate
  selector:
      matchLabels:
         app: kubia
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
       - image: luksa/kubia:v3
         name: nodejs
         readinessProbe: #httpGet类型的就绪探针
            periodSeconds: 1 #每秒探测一次
            httpGet: #get访问8080端口的根路径
               path: /
               port: 8080

---
#一个yaml文件可以定义多种资源,中间用---隔开

apiVersion: v1
kind: Service
metadata:
  name: kubia-dm-svc-v1

spec:
  type: NodePort
  selector:
    app: kubia

  ports:
  - port: 80
    targetPort: 8080
    nodePort: 32014

```


更新deployment配置文件

```
[root@k8s-master ~]# kubectl apply -f deployment-kubia-with-readinesscheck.yaml
deployment.apps/kubia-dm-v1 configured
service/kubia-dm-svc-v1 unchanged
```


查看更新状态,发现一直停留在等待第一个Pod就绪

```
[root@k8s-master ~]# kubectl rollout status deployment kubia-dm-v1
Waiting for deployment "kubia-dm-v1" rollout to finish: 1 out of 3 new replicas have been updated...

```

查看pods状态,发现3个旧版本的pod继续运行,新版本的Pod还未就绪


```
[root@k8s-master ~]# kubectl get pods
NAME                           READY   STATUS    RESTARTS   AGE
kubia-dm-v1-645b94954b-2gkw9   0/1     Running   0          49s
kubia-dm-v1-8585b844df-5rnx7   1/1     Running   0          53m
kubia-dm-v1-8585b844df-sz65w   1/1     Running   0          2m21s
kubia-dm-v1-8585b844df-ztd7x   1/1     Running   0          2m32s
```

> 因为maxUnavailable属性设置为0,所以不会删除任何旧版本的Pod.


默认情况下,在10分钟之内不能完成滚动升级的话,此次滚动升级被视为失败.

```
[root@k8s-master ~]# kubectl rollout status deployment kubia-dm-v1
Waiting for deployment "kubia-dm-v1" rollout to finish: 1 out of 3 new replicas have been updated...
error: deployment "kubia-dm-v1" exceeded its progress deadline
```

通过`kubectl describe deploy kubia-dm-v1`可以查看deployment状态.ProgressDeadlineExceeded表示升级超时

```
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    False   ProgressDeadlineExceeded
  
```

> 可以通过设定deployment.sepc.progressDeadlineSeconds来指定超时时间


##### 取消出错版本的滚动升级


```
[root@k8s-master ~]# kubectl rollout undo deployment kubia-dm-v1
deployment.apps/kubia-dm-v1 rolled back
[root@k8s-master ~]#
```

---

## 命令总结


```
#手动升级ReplicationController
kubectl rolling-update 当前rc名 新rc名 --image=新镜像名

#创建deployment资源,记录版本信息
kubectl create -f deployment配置文件 --record

#查看deployment滚动升级状态
kubectl rollout status deployment deployment名字

#修改deployment资源配置
kubectl patch deployment deployment名字 -p '{}'

#修改deployment配置的镜像信息
kubectl set image deployment deployment名字 镜像名=新镜像

#deployment回滚到上一个版本.在滚动升级超时时,用于取消滚动升级任务
kubectl rollout undo deployment deployment名字

#显示deployment回滚历史
kubectl rollout history deployment deployment名字

#deployment回滚到指定版本
kubectl rollout undo deployment deployment名字 --to-revision=版本号

#deployment暂停滚动升级
kubectl rollout pause deployment deployment名字

#deployment恢复滚动升级
kubectl rollout resume deployment deployment名字

#等待pod就绪时间
minReadySeconds

```


