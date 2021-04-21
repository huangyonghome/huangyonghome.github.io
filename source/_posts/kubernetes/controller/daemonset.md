---
title: Kubernetes DaemonSet
date: 2021-02-06 23:39:58
tags: k8s
categories: [kubernetes,controller]
comments: true
copyright: true
---



## DaemonSet

### 介绍

顾名思义,DaemonSet的主要作用是让你在kubernetes集群里运行一个Daemon Pod.所以,这个Pod有如下三个特征:

1.这个Pod运行在kubernetes集群的每一个节点(Node)上

2.每个节点只有一个这样的Pod实例

3.当有新的节点加入 Kubernetes 集群后，该 Pod 会自动地在新节点上被创建出来；而当旧节点被删除后，它上面的 Pod 也相应地会被回收掉.

这个机制听起来很简单，但 Daemon Pod 的意义确实是非常重要的。我随便给你列举几个例子：

1. 各种网络插件的 Agent 组件，都必须运行在每一个节点上，用来处理这个节点上的容器网络；
2. 各种存储插件的 Agent 组件，也必须运行在每一个节点上，用来在这个节点上挂载远程存储目录，操作容器的 Volume 目录；
3. 各种监控组件和日志组件，也必须运行在每一个节点上，负责这个节点上的监控信息和日志搜集。

<!--more-->

更重要的是，跟其他编排对象不一样，DaemonSet 开始运行的时机，很多时候比整个 Kubernetes 集群出现的时机都要早。

这个乍一听起来可能有点儿奇怪。但其实你来想一下：如果这个 DaemonSet 正是一个网络插件的 Agent 组件呢？

这个时候，整个 Kubernetes 集群里还没有可用的容器网络，所有 Worker 节点的状态都是 NotReady（NetworkReady=false）。这种情况下，普通的 Pod 肯定不能运行在这个集群上。所以，这也就意味着 DaemonSet 的设计，必须要有某种“过人之处”才行。

### DaemonSet工作原理

为了弄清楚 DaemonSet 的工作原理，我们还是按照老规矩，先从它的 API 对象的定义说起。下面是一个Nginx的daemonset资源清单

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx
  namespace: kube-system
  labels:
    k8s-app: nginx
spec:
  selector:
    matchLabels:
      name: nginx
  template:
    metadata:
      labels:
        name: nginx
    spec:
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      containers:
        - name: nginx
          image: nginx:1.19.6
          resources:
            limits:
              memory: 200Mi
            requests:
              cpu: 100m
              memory: 200Mi
```

这个DaemonSet非常简单,管理一个Nginx镜像的Pod.可以看到DaemonSet和Deployment非常相似,只不过没有replicas字段.他也使用selector选择管理所有携带了name=Nginx标签的POD

#### Daemonset创建Pod原理

那么，**DaemonSet 又是如何保证每个 Node 上有且只有一个被管理的 Pod 呢？**

显然，这是一个典型的“控制器模型”能够处理的问题。

DaemonSet Controller，首先从 Etcd 里获取所有的 Node 列表，然后遍历所有的 Node。这时，它就可以很容易地去检查，当前这个 Node 上是不是有一个携带了 name=fluentd-elasticsearch 标签的 Pod 在运行。

而检查的结果，可能有这么三种情况：

1. 没有这种 Pod，那么就意味着要在这个 Node 上创建这样一个 Pod；
2. 有这种 Pod，但是数量大于 1，那就说明要把多余的 Pod 从这个 Node 上删除掉；
3. 正好只有一个这种 Pod，那说明这个节点是正常的。

其中，删除节点（Node）上多余的 Pod 非常简单，直接调用 Kubernetes API 就可以了。

但是，**如何在指定的 Node 上创建新 Pod 呢？**



##### 1.**利用nodeAffinity**

如果你已经熟悉了 Pod API 对象的话，那一定可以立刻说出答案：用 nodeSelector，选择 Node 的名字即可。

```
nodeSelector:
    name: <Node 名字 >
```

没错。不过，在 Kubernetes 项目里，nodeSelector 其实已经是一个将要被废弃的字段了。因为，现在有了一个新的、功能更完善的字段可以代替它，即：nodeAffinity。我来举个例子：

```
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: metadata.name
            operator: In
            values:
            - node-geektime
```

在这个 Pod 里，我声明了一个 spec.affinity 字段，然后定义了一个 nodeAffinity。

而在这里，我定义的 nodeAffinity 的含义是：

1. requiredDuringSchedulingIgnoredDuringExecution：它的意思是说，这个 nodeAffinity 必须在每次调度的时候予以考虑。同时，这也意味着你可以设置在某些情况下不考虑这个 nodeAffinity；
2. 这个 Pod，将来只允许运行在“`metadata.name`”是“node-geektime”的节点上。

在这里，你应该注意到 nodeAffinity 的定义，可以支持更加丰富的语法，比如 operator: In（即：部分匹配；如果你定义 operator: Equal，就是完全匹配），这也正是 nodeAffinity 会取代 nodeSelector 的原因之一。

> 其实在大多数时候，这些 Operator 语义没啥用处。所以说，在学习开源项目的时候，一定要学会抓住“主线”。不要顾此失彼。

所以，**我们的 DaemonSet Controller 会在创建 Pod 的时候，自动在这个 Pod 的 API 对象里，加上这样一个 nodeAffinity 定义**。其中，需要绑定的节点名字，正是当前正在遍历的这个 Node。

当然，DaemonSet 并不需要修改用户提交的 YAML 文件里的 Pod 模板，而是在向 Kubernetes 发起请求之前，直接修改根据模板生成的 Pod 对象。这个思路，也正是我在前面讲解 Pod 对象时介绍过的。

创建刚才的Nginx的yaml清单:

```
[root@k8s-master daemonset]$kubectl get pods -n kube-system -l name=nginx
NAME          READY   STATUS    RESTARTS   AGE
nginx-b44l8   1/1     Running   0          66m
nginx-cvkgz   1/1     Running   0          66m
nginx-hldhl   1/1     Running   0          66m
```

查看其中任意一个pod的yaml文件

```
[root@k8s-master daemonset]$kubectl get pods -n kube-system nginx-b44l8 -o yaml
apiVersion: v1
kind: Pod
...略....
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchFields:
          - key: metadata.name
            operator: In
            values:
            - k8s-node2
...略....
```

这个是DaemonSet自动为Pod打上的nodeaffinity节点亲和性属性,表示将该Pod调度到`k8s-node2`这个hostname的节点上.

> 如果查看其它2个Nginx的Pod,他的nodeaffinity调度的节点名称自然也会不一样

**通过nodeaffinity,DaemonSet就可以确保每个Pod都调度到不同的k8s节点.而不会将多个pod调度到同一个节点.但是如果需要确保Pod可以被调度到节点上,还需要利用另外一个和调度相关的字段:tolerations**

##### 2.利用tolerations

tolerations(容忍度)这个字段意味着这个 Pod，会“容忍”（Toleration）某些 Node 的“污点”（Taint）。

而tolerations字段也是daemonset自动加上去的.还是执行上面的那条命令查看Pod的yaml清单文件

```
tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
  - effect: NoSchedule
    key: node.kubernetes.io/disk-pressure
    operator: Exists
  - effect: NoSchedule
    key: node.kubernetes.io/memory-pressure
    operator: Exists
  - effect: NoSchedule
    key: node.kubernetes.io/pid-pressure
    operator: Exists
  - effect: NoSchedule
    key: node.kubernetes.io/unschedulable
    operator: Exists
```

可以看到DaemonSet自动为这个Pod打上了很多容忍度,包括节点not-ready,unreachable,节点的磁盘,内存,pid的压力,以及哪怕节点被标记为`unschedulable` .就使得这些 Pod 可以忽略所有这些节点限制，继而保证每个节点上都会被调度一个 Pod。当然，如果这个节点有故障的话，这个 Pod 可能会启动失败，而 DaemonSet 则会始终尝试下去，直到 Pod 启动成功。

> 而在正常情况下，被标记了 unschedulable“污点”的 Node，是不会有任何 Pod 被调度上去的（effect: NoSchedule）

当然,你也可以在daemonset的资源清单里手动加上各种toleration污点容忍度.就像上面的例子:

```
tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
```

因为在默认情况下，Kubernetes 集群不允许用户在 Master 节点部署 Pod。因为，Master 节点默认携带了一个叫作`node-role.kubernetes.io/master`的“污点”。所以，为了能在 Master 节点上部署 DaemonSet 的 Pod，我就必须让这个 Pod“容忍”这个“污点”。

这时，你应该可以猜到，我在前面介绍到的**DaemonSet 的“过人之处”，其实就是依靠 Toleration 实现的。**

假如当前 DaemonSet 管理的，是一个网络插件的 Agent Pod，那么你就必须在这个 DaemonSet 的 YAML 文件里，给它的 Pod 模板加上一个能够“容忍”`node.kubernetes.io/network-unavailable`“污点”的 Toleration。正如下面这个例子所示：

```
...
template:
    metadata:
      labels:
        name: network-plugin-agent
    spec:
      tolerations:
      - key: node.kubernetes.io/network-unavailable
        operator: Exists
        effect: NoSchedule
```

在 Kubernetes 项目中，当一个节点的网络插件尚未安装时，这个节点就会被自动加上名为`node.kubernetes.io/network-unavailable`的“污点”。

**而通过这样一个 Toleration，调度器在调度这个 Pod 的时候，就会忽略当前节点上的“污点”，从而成功地将网络插件的 Agent 组件调度到这台机器上启动起来。**

### DaemonSet的滚动更新

通过命令查看daemonset的对象

```
[root@k8s-master daemonset]$kubectl get ds nginx -n kube-system
NAME    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
nginx   3         3         3       3            3           <none>          3h14m
```

就会发现 DaemonSet 和 Deployment 一样，也有 DESIRED、CURRENT 等多个状态字段。这也就意味着，DaemonSet 可以像 Deployment 那样，进行版本管理。这个版本，可以使用 kubectl rollout history 看到该daemonset的发布版本：

```
[root@k8s-master daemonset]$kubectl rollout history daemonset nginx -n kube-system
REVISION  CHANGE-CAUSE
1         <none>
```

接下来将nginx的镜像版本升级到1.19.0.顺便加上--record参数

```
[root@k8s-master daemonset]$kubectl set image ds nginx nginx=nginx:1.19.0 -n kube-system --record
```

观察升级过程

```
[root@k8s-master daemonset]$kubectl rollout status ds nginx -n kube-system
Waiting for daemon set "nginx" rollout to finish: 1 out of 3 new pods have been updated...
Waiting for daemon set "nginx" rollout to finish: 1 out of 3 new pods have been updated...
Waiting for daemon set "nginx" rollout to finish: 1 out of 3 new pods have been updated...
Waiting for daemon set "nginx" rollout to finish: 2 out of 3 new pods have been updated...
Waiting for daemon set "nginx" rollout to finish: 2 out of 3 new pods have been updated...
Waiting for daemon set "nginx" rollout to finish: 2 out of 3 new pods have been updated...
Waiting for daemon set "nginx" rollout to finish: 2 of 3 updated pods are available...
daemon set "nginx" successfully rolled out
```

在rollout history里就能看到滚动更新的记录:

```
[root@k8s-master daemonset]$kubectl rollout history daemonset nginx -n kube-system
daemonset.apps/nginx
REVISION  CHANGE-CAUSE
3         kubectl set image ds/nginx nginx=nginx:latest --record=true --namespace=kube-system
4         <none>
5         kubectl set image ds/nginx nginx=nginx:latest --record=true --namespace=kube-system
6         kubectl set image ds nginx nginx=nginx:1.19.0 --namespace=kube-system --record=true
```

通过后面的具体命令可以看到,这里我们是发布到了第6版.有了版本号，你也就可以像 Deployment 一样，将 DaemonSet 回滚到某个指定的历史版本了。

而我在前面的文章中讲解 Deployment 对象的时候，曾经提到过，Deployment 管理这些版本，靠的是“一个版本对应一个 ReplicaSet 对象”。可是，DaemonSet 控制器操作的直接就是 Pod，不可能有 ReplicaSet 这样的对象参与其中。**那么，它的这些版本又是如何维护的呢？**

所谓，一切皆对象！

在 Kubernetes 项目中，任何你觉得需要记录下来的状态，都可以被用 API 对象的方式实现。当然，“版本”也不例外。

Kubernetes v1.7 之后添加了一个 API 对象，名叫**ControllerRevision**，专门用来记录某种 Controller 对象的版本。比如，你可以通过如下命令查看 fluentd-elasticsearch 对应的 ControllerRevision：

```
[root@k8s-master daemonset]$kubectl get controllerrevision -n kube-system -l name=nginx
NAME               CONTROLLER             REVISION   AGE
nginx-66bcf54bbc   daemonset.apps/nginx   5          3h17m
nginx-6fdb467d8c   daemonset.apps/nginx   4          3h35m
nginx-757b664477   daemonset.apps/nginx   3          3h10m
nginx-7ccc97dc9f   daemonset.apps/nginx   6          4m30s
[root@k8s-master daemonset]$
```

可以看到每个版本号(REVISION)对应一个controller.而如果你使用 kubectl describe 查看这个 ControllerRevision 对象：

```
[root@k8s-master daemonset]$kubectl describe controllerrevision -n kube-system nginx-7ccc97dc9f
Name:         nginx-7ccc97dc9f
Namespace:    kube-system
Labels:       controller-revision-hash=7ccc97dc9f
              name=nginx
Annotations:  deprecated.daemonset.template.generation: 6
              kubernetes.io/change-cause: kubectl set image ds nginx nginx=nginx:1.19.0 --namespace=kube-system --record=true
API Version:  apps/v1
Data:
  Spec:
    Template:
      $patch:  replace
      Metadata:
        Creation Timestamp:  <nil>
        Labels:
          Name:  nginx
      Spec:
        Containers:
          Image:              nginx:1.19.0
          Image Pull Policy:  IfNotPresent
          Name:               nginx
          Resources:
            Limits:
              Memory:  200Mi
            Requests:
              Cpu:                     100m
              Memory:                  200Mi
          Termination Message Path:    /dev/termination-log
          Termination Message Policy:  File
        Dns Policy:                    ClusterFirst
        Restart Policy:                Always
        Scheduler Name:                default-scheduler
        Security Context:
        Termination Grace Period Seconds:  30
        Tolerations:
          Effect:  NoSchedule
          Key:     node-role.kubernetes.io/master
Kind:              ControllerRevision
Metadata:
  Creation Timestamp:  2021-03-30T06:00:05Z
  Owner References:
    API Version:           apps/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  DaemonSet
    Name:                  nginx
    UID:                   21e0bf42-536c-43c6-b961-f6cc51317eba
  Resource Version:        142550
  Self Link:               /apis/apps/v1/namespaces/kube-system/controllerrevisions/nginx-7ccc97dc9f
  UID:                     27f9800a-bafa-40d6-9a02-ca11fc2824ce
Revision:                  6
Events:                    <none>
```

就会看到，这个 ControllerRevision 对象，实际上是在 Data 字段保存了该版本对应的完整的 DaemonSet 的 API 对象。并且，在 Annotation 字段保存了创建这个对象所使用的 kubectl 命令。

接下来，我们可以尝试将这个 DaemonSet 回滚到 Revision=4 时的状态：

```
[root@k8s-master daemonset]$kubectl rollout undo daemonset nginx --to-revision=4 -n kube-system
daemonset.apps/nginx rolled back
```

这个 kubectl rollout undo 操作，实际上相当于读取到了 Revision=4 的 ControllerRevision 对象保存的 Data 字段。而这个 Data 字段里保存的信息，就是 Revision=1 时这个 DaemonSet 的完整 API 对象。

所以，现在 DaemonSet Controller 就可以使用这个历史 API 对象，对现有的 DaemonSet 做一次 PATCH 操作（等价于执行一次 kubectl apply -f “旧的 DaemonSet 对象”），从而把这个 DaemonSet“更新”到一个旧版本。

这也是为什么，在执行完这次回滚完成后，你会发现，DaemonSet 的 Revision 并不会从 Revision=6 退回到 4，而是会增加成 Revision=7。这是因为，一个新的 ControllerRevision 被创建了出来。

```
[root@k8s-master daemonset]$kubectl rollout history daemonset nginx -n kube-system
daemonset.apps/nginx
REVISION  CHANGE-CAUSE
3         kubectl set image ds/nginx nginx=nginx:latest --record=true --namespace=kube-system
5         kubectl set image ds/nginx nginx=nginx:latest --record=true --namespace=kube-system
6         kubectl set image ds nginx nginx=nginx:1.19.0 --namespace=kube-system --record=true
7         <none>
```

##### 商榷之处:

原文文档里说是`一个新的 ControllerRevision 被创建了出来` .但是经过实践发现,并没有创建一个新的revision=7的controllerrevision.而是仍然使用revision=4的controllerrevision,只不过将他的版本从4替代成了7..仔细对比下面回滚后的controllerrevision信息和回滚之前的信息可以发现这点:

##### 回滚到revision=4后,revision7出现了原本revision4的位置

```
[root@k8s-master daemonset]$kubectl get controllerrevision -n kube-system -l name=nginx
NAME               CONTROLLER             REVISION   AGE
nginx-66bcf54bbc   daemonset.apps/nginx   5          3h22m
nginx-6fdb467d8c   daemonset.apps/nginx   7          3h40m
nginx-757b664477   daemonset.apps/nginx   3          3h14m
nginx-7ccc97dc9f   daemonset.apps/nginx   6          9m26s
```

##### 下面这个是回滚到revision=4之前的controllervision信息.可以看到没有创建一个新的controllerrevision,而是原本revision=4的`nginx-6fdb467d8c`版本变更成了7

```
[root@k8s-master daemonset]$kubectl get controllerrevision -n kube-system -l name=nginx
NAME               CONTROLLER             REVISION   AGE
nginx-66bcf54bbc   daemonset.apps/nginx   5          3h17m
nginx-6fdb467d8c   daemonset.apps/nginx   4          3h35m
nginx-757b664477   daemonset.apps/nginx   3          3h10m
nginx-7ccc97dc9f   daemonset.apps/nginx   6          4m30s
[root@k8s-master daemonset]$
```

这可能是作者使用的k8s集群版本(1.11)和我实践的版本(1.17.3)不同

### 总结

相比于 Deployment，DaemonSet 只管理 Pod 对象，然后通过 nodeAffinity 和 Toleration 这两个调度器的小功能，保证了每个节点上有且只有一个 Pod。

与此同时，DaemonSet 使用 ControllerRevision，来保存和管理自己对应的“版本”。这种“面向 API 对象”的设计思路，大大简化了控制器本身的逻辑，也正是 Kubernetes 项目“声明式 API”的优势所在。

而且，相信聪明的你此时已经想到了，StatefulSet 也是直接控制 Pod 对象的，那么它是不是也在使用 ControllerRevision 进行版本管理呢？

没错。在 Kubernetes 项目里，ControllerRevision 其实是一个通用的版本管理对象。这样，Kubernetes 项目就巧妙地避免了每种控制器都要维护一套冗余的代码和逻辑的问题。

### 参考资料:

张磊---<深入剖析Kubernetes>: 容器化守护进程的意义：DaemonSet

