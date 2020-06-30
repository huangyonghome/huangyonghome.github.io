---
title: kubernetes 高级调度
date: 2020-06-26 11:59:58
tags: k8s
categories: kubernetes
comments: true
copyright: true
---



## kubernetes 高级调度

### 介绍

本章主要介绍pod的高级调度功能.主要涵盖2方面调度特性:

* 节点污染和pod容忍度
* 节点亲缘性

---

### 一.节点污染和pod容忍度

#### 概念

**节点污点**: 该特性用于限制哪些Pod可以被调度到某一个节点.

**pod容忍度**:当一个Pod容忍某个节点的污点,这个pod才能被调度到该节点

默认情况下k8s集群中的master主节点就设置了污点,.这样才能保证只有控制面板等系统组件才能部署在主节点上.应用pod只能被调度到工作节点

---

#### 显示节点污点信息

<!--more-->

通过`kubectl describe node`可以查看节点的污点信息.例如下面是master节点污点信息

```bash
[root@k8s-master ~]# kubectl describe node k8s-master
Taints:             node-role.kubernetes.io/master:NoSchedule
```

master节点包含一个污点(taints).污点包含一个key,value,effect.格式为:<key>=<value>:<effect>

上面的污点信息包含一个`node-role.kubernetes.io/master`的key.一个空的value,以及值为`NoSchedule`的effect.

这个污点将阻止pod调度到这个节点上,除非有pod能容忍这个污点.通常能容忍这个污点的Pod都是kubernetes系统级别的Pod.

例如观察一个Kube-system名称空间下的coredns系统级别的pod信息:

```bash
[root@k8s-master ~]# kubectl describe po coredns-7f9c544f75-9sh28 -n kube-system

Tolerations:     CriticalAddonsOnly
                 node-role.kubernetes.io/master:NoSchedule
                 node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
```

该pod容忍度(tolerations)包含了4个容忍度,其中`node-role.kubernetes.io/master:NoSchedule`容忍度显示该pod可以容忍master节点上的污点.从而该pod可以调度到master节点上

##### 污点效果

上面列出的pod展示了2种污点effect(污点效果):`NoSchedule`和`NoExecute`.每一个污点都可以关联一个effect.下面介绍各污点效果.

* **NoSechedule**: 如果pod没有容忍这些污点,这pod不能被调度到包含这个污点的节点
* **PreferNoSechedule** NoSechedule的宽松版本,表示尽量阻止pod被调度到这个节点.但是如果没有其他节点可以调度,pod依然会被调度到这个节点
* **NoExecute** 上面两者只在调度期间起作用,而NoExecute也会影响正在节点上运行中的pod.如果在一个节点上添加了NoExecute污点.那么在这个节点上正在运行的pod如果没有容忍这个污点,会被这个节点驱除

---

#### 添加自定义污点

如果一个单独K8s集群上同时有生产环境和非生产环境的应用.那么可以在生产环境上添加自定义污点来防止其他环境的Pod调度到生产节点.

添加污点命令:`kubectl taint`

命令格式: `kubectl taint node NodeName key=value:effect`

例如下面命令将node1的节点添加一个key为`node-type`.value为`production`.效果为`NoSchedule`的污点

```
root@k8s-master ~]# kubectl taint node k8s-node1 node-type=production:NoSchedule
node/k8s-node1 tainted
```

下面部署一个常规的多副本pod.此时所有pod都被调度到k8s-node2节点上.无法调度到node1

```bash
[root@k8s-master ~]# kubectl run test --image busybox --replicas 5 -- sleep 999

[root@k8s-master ~]# kubectl get pods -o wide  
test-69c6778cfb-2jgbn           1/1     Running   0          4m38s   10.100.169.188   k8s-node2   <none>           <none>
test-69c6778cfb-7lbnw           1/1     Running   0          4m38s   10.100.169.187   k8s-node2   <none>           <none>
test-69c6778cfb-c4vrx           1/1     Running   0          4m38s   10.100.169.190   k8s-node2   <none>           <none>
test-69c6778cfb-grkhc           1/1     Running   0          4m38s   10.100.169.185   k8s-node2   <none>           <none>
test-69c6778cfb-xc2sp           1/1     Running   0          4m38s   10.100.169.130   k8s-node2   <none>           <none>
```

---

#### 在pod上添加污点容忍度

如果想让pod部署到node1这个已经打了`NoSchedule`污点标签的节点上.那么Pod必须声明能匹配该节点污点的容忍度.下面是一个例子

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod
spec:
  selector:
    matchLabels:
       node-type: production
  replicas: 5
  template:
    metadata:
       name: taint-kubia-v1
       labels:
         node-type: production
    spec:
      containers:
           - image: busybox
             command: ["sleep"]
             args: ["999999"]
             name: kubia-busybox
      tolerations:
        - key: node-type
          operator: Equal
          value: production
          effect: NoSchedule
```

在pod的`tempalte.spec.tolerations`属性下定义污点key.该key的值匹配(Equal)node1节点上的污点.所以该pod能够被调度到node1节点

```bash
[root@k8s-master ~]# kubectl get pods -o wide
NAME                            READY   STATUS    RESTARTS   AGE    IP               NODE        NOMINATED NODE   READINESS GATES
prod-59b4554db6-d24d5           1/1     Running   0          24s    10.100.169.191   k8s-node2   <none>           <none>
prod-59b4554db6-kmzj2           1/1     Running   0          24s    10.100.36.114    k8s-node1   <none>           <none>
prod-59b4554db6-nlc8s           1/1     Running   0          24s    10.100.36.116    k8s-node1   <none>           <none>
prod-59b4554db6-pvzrw           1/1     Running   0          24s    10.100.36.115    k8s-node1   <none>           <none>
prod-59b4554db6-t8n5c           1/1     Running   0          24s    10.100.169.131   k8s-node2   <none>           <none>
```

---

#### 在node节点添加污点,驱逐正在该节点上运行的pod

当前在node2节点上运行了以下pod

```bash
[root@k8s-master ~]# kubectl get pods -o wide  | grep k8s-node2
cloud-busybox-99c65b774-6bbbc   1/1     Running   80         3d8h    10.100.169.183   k8s-node2   <none>           <none>
cloud-busybox-99c65b774-zcw78   1/1     Running   80         3d8h    10.100.169.182   k8s-node2   <none>           <none>
curl-custome-sa                 2/2     Running   0          8d      10.100.169.177   k8s-node2   <none>           <none>
prod-59b4554db6-d24d5           1/1     Running   4          9m49s   10.100.169.191   k8s-node2   <none>           <none>
prod-59b4554db6-t8n5c           1/1     Running   4          9m49s   10.100.169.131   k8s-node2   <none>           <none>
statefulset-kubia-v1-1          1/1     Running   0          10d     10.100.169.172   k8s-node2   <none>           <none>
test-69c6778cfb-2jgbn           1/1     Running   80         22h     10.100.169.188   k8s-node2   <none>           <none>
test-69c6778cfb-7lbnw           1/1     Running   80         22h     10.100.169.187   k8s-node2   <none>           <none>
test-69c6778cfb-c4vrx           1/1     Running   80         22h     10.100.169.190   k8s-node2   <none>           <none>
test-69c6778cfb-grkhc           1/1     Running   80         22h     10.100.169.185   k8s-node2   <none>           <none>
test-69c6778cfb-xc2sp           1/1     Running   80         22h     10.100.169.130   k8s-node2   <none>           <none>
```

如果想要在node2节点上打上污点信息,只允许non-production环境的pod才能调度到该节点上.可以直接打上污点标签

```bash
[root@k8s-master ~]# kubectl taint node k8s-node2 node-type=non-production:NoExecute
node/k8s-node2 tainted

#此时该节点上的所有pod都被终止
[root@k8s-master ~]# kubectl get pods -o wide  | grep k8s-node2
cloud-busybox-99c65b774-6bbbc   1/1     Terminating         80         3d8h   10.100.169.183   k8s-node2   <none>           <none>
cloud-busybox-99c65b774-zcw78   1/1     Terminating         80         3d8h   10.100.169.182   k8s-node2   <none>           <none>
curl-custome-sa                 2/2     Terminating         0          8d     10.100.169.177   k8s-node2   <none>           <none>
statefulset-kubia-v1-1          1/1     Terminating         0          10d    10.100.169.172   k8s-node2   <none>           <none>
test-69c6778cfb-2jgbn           1/1     Terminating         80         22h    10.100.169.188   k8s-node2   <none>           <none>
test-69c6778cfb-7lbnw           1/1     Terminating         80         22h    10.100.169.187   k8s-node2   <none>           <none>
test-69c6778cfb-c4vrx           1/1     Terminating         80         22h    10.100.169.190   k8s-node2   <none>           <none>
test-69c6778cfb-grkhc           1/1     Terminating         80         22h    10.100.169.185   k8s-node2   <none>           <none>
test-69c6778cfb-xc2sp           1/1     Terminating         80         22h    10.100.169.130   k8s-node2   <none>           <none>

#所有pod都被调度到node1节点.(注意有一个运行在Node2上的statefulset类型的Pod被终止了,由于statefulset的特性,该pod不会被自动调度到其他节点,除非手动强制删除)
[root@k8s-master ~]# kubectl get pods -o wide
NAME                            READY   STATUS        RESTARTS   AGE     IP               NODE        NOMINATED NODE   READINESS GATES
cloud-busybox-99c65b774-j5ggn   0/1     Pending       0          144m    <none>           <none>      <none>           <none>
cloud-busybox-99c65b774-ksj8x   1/1     Running       82         3d10h   10.100.36.103    k8s-node1   <none>           <none>
cloud-busybox-99c65b774-qtzv9   0/1     Pending       0          144m    <none>           <none>      <none>           <none>
prod-6d8fdb7654-9xlqq           1/1     Running       0          7m23s   10.100.36.109    k8s-node1   <none>           <none>
prod-6d8fdb7654-cwsfp           1/1     Running       0          7m23s   10.100.36.108    k8s-node1   <none>           <none>
prod-6d8fdb7654-gbc79           1/1     Running       0          7m10s   10.100.36.119    k8s-node1   <none>           <none>
prod-6d8fdb7654-lsjc4           1/1     Running       0          7m15s   10.100.36.110    k8s-node1   <none>           <none>
prod-6d8fdb7654-mcmt7           1/1     Running       0          7m23s   10.100.36.111    k8s-node1   <none>           <none>
statefulset-kubia-v1-0          1/1     Running       0          10d     10.100.36.101    k8s-node1   <none>           <none>
statefulset-kubia-v1-1          0/1     Terminating   0          10d     10.100.169.172   k8s-node2   <none>           <none>
statefulset-kubia-v1-2          1/1     Running       0          10d     10.100.36.97     k8s-node1   <none>           <none>
statefulset-kubia-v1-3          1/1     Running       0          10d     10.100.36.99     k8s-node1   <none>           <none>

```

---

#### 了解污点和污点容忍度使用场景

​       节点可以拥有多个污点信息,而pod也可以有多个污点容忍度.污点容忍度可以通过操作符`Equal`来匹配一个污点信息,也可以通过`Exists`操作符来匹配污点的key

---

### 二.节点亲缘性(node affinity)

污点可以让pod远离特定的节点.节点亲缘性是一种更新的机制.这种机制允许Kubernetes将pod只调度到某个节点上面

与节点选择器类似,每个pod可以定义自己的节点亲缘性规则,这些规则可以允许你指定硬性限制或者偏好.如果指定一种偏好的话,你将告知Kubernetes对于某个特定的pod.它更倾向于调度到某些节点上.如果没法实现的话,pod将被调度到其他节点.

##### 检查默认node节点标签:

```bash
[root@k8s-master ~]# kubectl describe node k8s-node2
Name:               k8s-node2
Roles:              <none>
#Labels表示该节点拥有的默认的标签.
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=k8s-node2
                    kubernetes.io/os=linux
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    projectcalico.org/IPv4Address: 172.16.20.253/24
                    projectcalico.org/IPv4IPIPTunnelAddr: 10.100.169.128
                    volumes.kubernetes.io/controller-managed-attach-detach: true
```

##### 给节点打一个标签.

```
#给node1节点打一个标签.disk=ssd
[root@k8s-master ~]# kubectl label node k8s-node1 disk=ssd
```

##### 回顾nodeSelector节点选择器

```yaml
#下面是一个deployment的配置文件,Pod指定了标签选择器
[root@k8s-master ~]# cat node-affinity-nodeslector.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubia-nodeselector-ssd
spec:
  replicas: 4
  selector:
     matchLabels:
        app:  kubia-nodeselector
  template:
    metadata:
      name: kubia-nodeselector-ssd
      labels:
         app:  kubia-nodeselector

    spec:
      nodeSelector:
         disk: ssd
      containers:
        - image: luksa/kubia
          name: kubia-nodeselector-ssd

```

所有节点都调度到符合这个标签的node1节点

```bash
[root@k8s-master ~]# kubectl get pods -o wide | grep kubia-nodeselector-ssd
kubia-nodeselector-ssd-7d5b94f9bd-cqmsv   1/1     Running       0          2m2s    10.100.36.124    k8s-node1   <none>           <none>
kubia-nodeselector-ssd-7d5b94f9bd-djkxr   1/1     Running       0          2m2s    10.100.36.125    k8s-node1   <none>           <none>
kubia-nodeselector-ssd-7d5b94f9bd-htggw   1/1     Running       0          2m2s    10.100.36.126    k8s-node1   <none>           <none>
kubia-nodeselector-ssd-7d5b94f9bd-vm2rv   1/1     Running       0          2m2s    10.100.36.127    k8s-node1   <none>           <none>

```

---

#### 节点亲缘性配置

将上面的节点标签选择器换成用节点亲缘性来表达

```yaml
[root@k8s-master ~]# cat  node-affinity-disk-ssd.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubia-nodeaffinity-ssd
spec:
  replicas: 4
  selector:
     matchLabels:
        app:  kubia-nodeaffinity
  template:
    metadata:
      name: kubia-nodeaffinity
      labels:
         app:  kubia-nodeaffinity

    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
             nodeSelectorTerms:
               - matchExpressions:
                    - key: disk
                      operator: In
                      values:
                         - ssd
      containers:
        - image: luksa/kubia
          name: kubia-nodeselector-ssd

```

第一印象是节点亲缘性写法比节点选择器要复杂的多(这是因为节点亲缘性的表达能力更强).

##### 节点亲缘性属性

pod的`affinity`属性配置了节点亲缘性相关信息.该属性下有以下几种亲缘性:

* **nodeAffinity** 顾名思义,节点亲缘性调度规则
* **podAffinity**  Pod亲缘性调度规则,例如某类节点调度到同一个节点,地域,可用区
* **podAntiAffinity** 和podAffinity完全相反,pod反亲缘性调度规则,

在这个例子中石油了**nodeAffinity**表示节点亲缘性规则.在这个规则下有一个极其长的属性.把这个属性的名字分成两段,然后分别看下他们的含义:

* **requiredDuringScheduling** 表明该字段下定义的规则.为了让pod能够调度到该节点,明确指出该节点必须包含的标签.重点包含以下2个条件.也就是说在pod调度期间,节点必须要包含标签:
  - **required**(必须),
  - **DuringScheduling**(在pod调度期间)
* ...**IgnoredDuringExecution** 表示该规则不会影响已经在节点上运行的pod.重点包含以下2个条件.也就是说无论节点是否具备某个标签,或者无论pod怎么调度,都不会影响已经运行的Pod
  - **Ignored**(忽略)
  - **DuringExecution**(在pod已运行期间)

接下来的字段属性比较容易理解,

* **nodeSelectorTerms**: 对象类型,表示一组节点选择器项列表
* **matchExpressions**: 匹配表达式,也是一个对象类型.包含以下属性
  - **key**: 必要字段.
  - **operator**: 必要字段,key和value关系的表达式,有**In**,**NotIn**,**Exists**,**DoesNotExist**,**Gt**,**Lt**
  - **values**: key的值,是一个字符串的数组格式.

> 通过`kubectl explain pods.spec.affinity`可以查询属性字段的含义以及配置用法

---

### pod调度节点优先

节点亲缘性还有一个属性`preferredDuringSchedulingIgnoredDuringExecution`这个属性字段指定调度器优先考虑哪些节点.

想象一下如果你在阿里云同一地域下的多个可用区都有部署节点服务器,你想要将Pod优先部署在zone1,并且是部署在专有服务器上.如果zone1下没有足够的服务器资源,那么部署到其他可用区(zone2)也是可以接受的.那么节点亲缘性就可以实现这种功能.

##### 给节点加上可用区等信息标签

下面的配置中给2个节点加上标签,模拟他们在不同的可用区服务器,以及属于不同的服务器类型(专用和共享)

```bash
[root@k8s-master ~]# kubectl label node k8s-node1 zone=zone1
node/k8s-node1 labeled
[root@k8s-master ~]# kubectl label node k8s-node1 share-type=dedicated
node/k8s-node1 labeled
[root@k8s-master ~]# kubectl label node k8s-node2 share-type=shared
node/k8s-node2 labeled
[root@k8s-master ~]# kubectl label node k8s-node2 zone=zone2
node/k8s-node2 labeled
[root@k8s-master ~]#
```

##### 指定优先级节点亲缘性规则

下面创建一个deployment资源,pod节点指定了优先调度到zone1区域(权重80),次优先调度到dedicated服务器(权重20)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-affinity-weight
spec:
  replicas: 10
  selector:
    matchLabels:
       app: node-affinity-weight
  template:
    metadata:
      name: node-affinity-weight
      labels:
         app: node-affinity-weight
    spec:
      containers:
        - image: luksa/kubia
          name: kubia
      affinity:
        nodeAffinity:
          #优先调度到某个节点,注意不是必须要调度到该节点
          preferredDuringSchedulingIgnoredDuringExecution:
             #权重越高,优先级越高,优先调度到zone1
              - weight: 80
                preference:
                  matchExpressions:
                   - key: zone
                     operator: In
                     values:
                        - zone1

             #如果以上需求不能满足,则调度到满足以下标签的节点
              - weight: 20
                preference:
                  matchExpressions:
                    - key: share-type
                      operator: In
                      values:
                        - dedicated
```

此时大部分Pod都优先部署到node1节点.之所以不是所有Pod都被调度到Node1节点是因为除了节点亲缘性优先级外还有其他的优先级函数来决定节点被调度到哪.

```bash
[root@k8s-master ~]# kubectl get pods -o wide
NAME                                    READY   STATUS    RESTARTS   AGE   IP               NODE        NOMINATED NODE   READINESS GATES
node-affinity-weight-7777f464d6-8h9qq   1/1     Running   0          33s   10.100.36.75     k8s-node1   <none>           <none>
node-affinity-weight-7777f464d6-8xchv   1/1     Running   0          33s   10.100.169.132   k8s-node2   <none>           <none>
node-affinity-weight-7777f464d6-bnsvs   1/1     Running   0          33s   10.100.36.79     k8s-node1   <none>           <none>
node-affinity-weight-7777f464d6-bxbpb   1/1     Running   0          33s   10.100.36.71     k8s-node1   <none>           <none>
node-affinity-weight-7777f464d6-bznfw   1/1     Running   0          33s   10.100.36.78     k8s-node1   <none>           <none>
node-affinity-weight-7777f464d6-cgdfq   1/1     Running   0          33s   10.100.36.77     k8s-node1   <none>           <none>
node-affinity-weight-7777f464d6-flvcf   1/1     Running   0          33s   10.100.169.135   k8s-node2   <none>           <none>
node-affinity-weight-7777f464d6-pb666   1/1     Running   0          33s   10.100.169.134   k8s-node2   <none>           <none>
node-affinity-weight-7777f464d6-tqd5k   1/1     Running   0          33s   10.100.36.74     k8s-node1   <none>           <none>
node-affinity-weight-7777f464d6-vpknb   1/1     Running   0          33s   10.100.36.76     k8s-node1   <none>           <none>
```

----

### pod硬亲缘性

上一节已经了解节点亲缘性规则是如何影响Pod被调度到哪个节点的.这些规则只影响了Pod和节点之前的亲缘性.但是有时候也希望有能力指定pod自身之间的亲缘性.

例如,有一个前端pod,和一个后端Pod,我们期望将前端Pod部署到和后端pod同一个节点,降低前后端pod之间的访问延时,提高应用性能.使用Pod亲缘性可以满足这种场景的需求.

>  pod亲缘性和节点亲缘性区别是.pod亲缘性不关心pod被调度到具体哪个节点,而是只要求被调度到其他某组pod同一个节点上.

在下面的例子中部署一个1后端pod.和5个前端pod.使用节点亲缘性将5个前端Pod部署到后端pod相同的节点上

##### 部署一个后端pod

```
#启动一个backend的pod.指定一个标签
[root@k8s-master ~]# kubectl run backend -l app=backend --image busybox -- sleep 999999
```

##### 部署前端Pod亲缘性

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-affinity

spec:
  replicas: 5
  selector:
    matchLabels:
       app: frontend

  template:
    metadata:
      name: frontend
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: luksa/kubia
      affinity:
        #节点亲缘性
        podAffinity:
         #一个required强制性要求而不是prefered
          requiredDuringSchedulingIgnoredDuringExecution:
               #此pod必须被调度到满足以下条件的节点.该节点运行了app:backend标签的Pod
             - topologyKey: kubernetes.io/hostname
               #满足以下标签选择器的pod
               labelSelector:
                  matchLabels:
                      app: backend
```

代码清单显示要求将pod调度到和其他包含app=backend标签的Pod所在的相同节点上(通过topologyKey字段指定)

> 除了使用matchLabels字段外,也可以使用表达能力更强的matchExpressions字段

此时所有的前端pod都被部署到后端Pod同一个节点,没有被部署到其他节点(node1)

```bash
[root@k8s-master ~]# kubectl get pods -o wide
NAME                            READY   STATUS    RESTARTS   AGE     IP               NODE        NOMINATED NODE   READINESS GATES
backend-7d4c66b6b5-4mhpl        1/1     Running   0          43m     10.100.169.133   k8s-node2   <none>           <none>
pod-affinity-694cf9455c-6h2dq   1/1     Running   0          2m46s   10.100.169.139   k8s-node2   <none>           <none>
pod-affinity-694cf9455c-jtmd8   1/1     Running   0          2m46s   10.100.169.140   k8s-node2   <none>           <none>
pod-affinity-694cf9455c-p6m7d   1/1     Running   0          2m46s   10.100.169.141   k8s-node2   <none>           <none>
pod-affinity-694cf9455c-qbr4c   1/1     Running   0          2m46s   10.100.169.137   k8s-node2   <none>           <none>
pod-affinity-694cf9455c-vfp84   1/1     Running   0          2m46s   10.100.169.138   k8s-node2   <none>           <none>
```

> 有趣的是,如果现在删除了后端Pod,调度器会将后端pod还是调度到node2.即便后端pod没有定义任何Pod亲缘性规则.
>
> 这种设置很合理,因为如何后端Pod被调度到其他节点,前端pod的亲缘性规则就被打破了

##### topologyKey

在上面的例子中topology属性为kubernetes.io/hostname.除此之外还有其他属性:

* **failure-domain.beta.kubernetes.io/zone** 云服务提供商不同的不同可用区
* **failure-domain.beta.kubernetes.io/region** 云服务提供商不同的不同地域
* 自定义键

---

### pod软亲缘性

pod软亲缘性稍微的区别就是没有硬性要求一定要部署到某个节点.拿前面一个例子来说,软亲缘性只是优先将前端pod调度到和后端Pod相同的节点,但是如果条件不满足要求,也可以调度到其他节点

软亲缘性的配置文件和硬亲缘性的主要区别就在于.**required**开头的属性是硬性要求.**preferred**是软要求

下面将之前的前端pod的deployment删除,然后部署下面的软podAffinity

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-affinity-frontend-preferred
spec:
  replicas: 5
  selector:
    matchLabels:
       app: frontend

  template:
     metadata:
       name: pod-affinity-frontend-preferred
       labels:
         app: frontend
     spec:
       containers:
         - name: frontend
           image: luksa/kubia
       affinity:
          podAffinity:
             preferredDuringSchedulingIgnoredDuringExecution:
                - weight: 80
                  podAffinityTerm:
                      topologyKey: kubernetes.io/hostname
                      labelSelector:
                        matchExpressions:
                            - key: app
                              operator: In
                              values:
                                - backend
```

此时看到几乎所有的前端pod都部署到和后端Pod相同的节点上.但是其中还是有一个pod被部署到node1.

正如**nodeAffinity**的软亲缘性一样.kubernetes不会将所有的Pod都完全按照要求部署到同一个节点,还有其他优先调度函数在起作用,将pod调度到其他节点.

这样的设置是有道理的,因为Kubernets考虑到如果该节点出现宕机故障会导致整个应用不可用.

```bash
[root@k8s-master ~]# kubectl get pods -o wide
NAME                                               READY   STATUS    RESTARTS   AGE   IP               NODE        NOMINATED NODE   READINESS GATES
backend-7d4c66b6b5-hvkph                           1/1     Running   0          41m   10.100.169.142   k8s-node2   <none>           <none>
pod-affinity-frontend-preferred-7b7b9fb784-4jc9r   1/1     Running   0          17s   10.100.36.80     k8s-node1   <none>           <none>
pod-affinity-frontend-preferred-7b7b9fb784-bd97q   1/1     Running   0          17s   10.100.169.146   k8s-node2   <none>           <none>
pod-affinity-frontend-preferred-7b7b9fb784-rfgp7   1/1     Running   0          17s   10.100.169.145   k8s-node2   <none>           <none>
pod-affinity-frontend-preferred-7b7b9fb784-s7qtc   1/1     Running   0          17s   10.100.169.143   k8s-node2   <none>           <none>
pod-affinity-frontend-preferred-7b7b9fb784-skhhq   1/1     Running   0          17s   10.100.169.144   k8s-node2   <none>           <none>
```

---

### pod非亲缘性

和pod亲缘性完全相反,pod非亲缘性让两组pod远离彼此,不被调度到同一个节点.这种场景比较常见.例如当2个集合的pod调度到同一个节点上会影响彼此的性能.

pod非亲缘性和pod亲缘性配置几乎一模一样,唯一的区别就是podAntiAffinity替代podAffinity

删除之前的例子,使用下面的Pod配置文件

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-antiaffinity

spec:
  replicas: 5
  selector:
    matchLabels:
       app: frontend

  template:
    metadata:
      name: frontend
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: luksa/kubia
      affinity:
        #节点亲缘性
        podAntiAffinity:
         #一个required强制性要求而不是prefered
          requiredDuringSchedulingIgnoredDuringExecution:
               #此pod必须不被调度到运行了app:backend标签Pod的节点
             - topologyKey: kubernetes.io/hostname
               #满足以下标签选择器的pod
               labelSelector:
                  matchLabels:
                      app: backend
```

前端的所有pod都被调度到和后端pod不一样的节点

```bash
[root@k8s-master ~]# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE    IP               NODE        NOMINATED NODE   READINESS GATES
backend-7d4c66b6b5-hvkph            1/1     Running   0          168m   10.100.169.142   k8s-node2   <none>           <none>
pod-antiaffinity-5b9fd9846c-k5flw   1/1     Running   0          61s    10.100.36.81     k8s-node1   <none>           <none>
pod-antiaffinity-5b9fd9846c-t4l8z   1/1     Running   0          61s    10.100.36.89     k8s-node1   <none>           <none>
pod-antiaffinity-5b9fd9846c-wp5mf   1/1     Running   0          61s    10.100.36.83     k8s-node1   <none>           <none>
pod-antiaffinity-5b9fd9846c-xkqhm   1/1     Running   0          61s    10.100.36.82     k8s-node1   <none>           <none>
pod-antiaffinity-5b9fd9846c-xzg7t   1/1     Running   0          61s    10.100.36.84     k8s-node1   <none>           <none>
```

pod软非亲缘性就不再介绍了

---

### 总结

* 如果节点上添加了1个污点信息.除非Pod容忍这些污点,否则pod不会被调度到该节点
* 有3种类型的污点:`Noschdule`完全阻止pod调度,`PreferNoSchedule`不完全阻止,`NoExecute`将已经在运行的Pod从节点驱逐
* `NoExecute`污点的节点可以设置等待时间,当节点不可用时,pod重新调度时,最长等待时间
* 节点亲缘性允许指定pod应该被调度到哪些节点.有硬性(**required**)要求,也有软性优先级(**preferred**)
* pod亲缘性用于将pod调度到和另一个pod相同的一个节点.(基于pod的标签)
* pod亲缘性的topologyKey属性表示了被调度的pod和另一组pod的距离.(在同一个节点,同一个机柜,同一个可用区,或者同一个地域)
* pod非亲缘性和亲缘性相反,用于将pod调度到远离某组pod的节点
* 无论是节点亲缘性还是pod亲缘性可以设置是硬性要求还是软性优选选择