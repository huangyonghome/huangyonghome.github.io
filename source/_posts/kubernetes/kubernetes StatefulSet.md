---
title: kubernetes StatefulSet
date: 2020-06-26 11:59:58
tags: k8s
categories: kubernetes
comments: true
copyright: true
---

## kubernetes StatefulSet

### 介绍

​     我们之前学习过ReplicaController(RC),ReplicaSet(RS),Deployment(deploy)等,这些资源均可以通过一个Pod模板创建多个Pod副本.这些pod除了IP和名字不一样外,其他均一模一样.

​     如果pod模板关联到特定的持久卷声明,那么这些pod都共享同一个存储.这些pod是完全一样,也无所谓运行在哪个节点上.可以任何删除和替换.

​     但是在实际场景中,并不是所有的应用都满足这样的要求.比如主从关系,主备关系,数据存储类应用等.这些场景每个pod都会在本地保留一份数据,而且与其他Pod有数据对应关系.如果pod一旦被删除,即便新创建个pod出来,实例之间的对应关系也会失败,从而导致应用失败.

   为了支持有状态的应用,Kubernetes在Deployment的基础上扩展出了StatefulSet资源

---

### StatefulSet 特点

statefulSet的特点即是有状态的应用特点:

1. 稳定且需要唯一的网络标识符;

   * 这要求pod的主机名和IP地址永久不变,即使删除了一个pod,新创建的Pod也必须继承前一个Pod的主机标识符


2. 稳定且持久的存储;

   * 可实现持久存储,新增或者减少pod,存储不会随之变化,并且删除一个pod时,关联到此pod的存储不会随之删除

3. 要求有序,平滑的部署和扩展
   * 在Mysql等集群,要先启动主节点,然后启动从节点,第二从节点等等

4. 要求有序,平滑的终止和删除
   * 同样,应用的终止和删除也是有顺序的,按照启动的逆序进行.例如Mysql启动时,先启动主节点,再启动从节点.终止的时候,先关闭从节点,再关闭主节点

5. 有序的滚动更新
   * 在Mysql更新时,也应该先更新所有从节点.最后更新主节点

<!--more-->

---

   ### StatefulSet 依赖组件

结合以上的特点,StatefulSet依赖以下组件:

* headless service(无头服务): 用于DNS发现各pod网络地址
* pv存储卷: 底层存储卷
* volumeClaimTemplate(PVC申请模板): 用于每个Pod申请独立的存储卷

---

###   headless Service

​     我们以前学习过,Service是Kubernetes项目中用来将一组Pod暴露给外界访问的一种机制.外部客户端通过Service地址可以随机访问到某个具体的Pod

​     之前学过几种Service类型,包括nodeport,loadbalancer等等.所有这类Service都有一个VIP(虚拟IP),访问Service VIP,Service会将请求转发到后端的Pod上,

​     还有一种Service是Headless Service(无头服务),这类Service自身不需要VIP,当DNS解析该Service时,会解析出Service后端的Pod地址.这样设置的好处是Kubernetes项目为Pod分配唯一的"可解析身份",只要知道一个pod的名字和对应的Headless Service名字,就可以通过这条DNS访问到后端的Pod

---

###  持久存储

我们知道通过headless Service使Pod有一个稳定的网络标识,那么存储呢?有状态的应用必须有自己独立的存储,即便这个pod被删除,新创建出来的pod(新pod与旧pod拥有相同的网络表示)也必须挂载相同的存储.

​      之前在学习kubernetes的存储时,我们学习过PV,PVC存储卷,通过pod模板关联一个持久卷声明就可以为pod提供一个持久卷存储.因为持久卷声明(PVC)和持久卷(PV)是一对一关系.但是之前接触过的ReplicationController,ReplicaSet,Deployment等资源创建的pod是同一个模板创建的,所以共享的是同一个持久卷存储.而StatefulSet要求每个pod都需要有独立的持久卷声明和存储.所以StatefulSet要求关联到一个或多个不同的持久卷声明模板.这些持久卷声明会在pod创建之前准备就绪,并且关联到每个pod中.

---

### 持久卷的创建和删除

​      扩容一个StatefulSet副本时,会创建2个或者多个对象: pod实例已经与之关联的一个或者多个持久卷声明.但是当StatefulSet缩容时,只会删除一个Pod,而留下持久卷声明.这就意味着删除Pod时,与pod关联的持久卷存储数据并不会被删除.如果持久卷声明被手动删除,那么持久卷上的数据则会消失.

​     因为缩容会保留持久卷声明,所以在随后的扩容操作中,新的pod实例会使用绑定在持久卷上相同的声明和其上的数据.所以如果因为误操作而缩容一个StatefulSet副本后,可以做一次扩容操作,新的pod实例会运行到与之前完全一致的状态,甚至连pod名字也是一样的

---

### 部署StatefulSet应用

部署StatefulSet应用之前,需要创建几个不同类型的对象.

* 一个演示用的docker镜像

* 存储数据文件的持久卷(PV)

* 一个Headless Service服务实例

* Statefulset模板

#### 准备一个docker镜像

这里使用书上提供的luksa/kubia-pet镜像,这个镜像是一个Node应用,当应用接收到一个POST请求时,将请求中的body写入到某个文件,当接收到一个GET请求时,返回pod主机名以及改文件中的内容.



#### 创建持久化存储卷(pv)

因为稍后会调度StatefulSet创建3个副本.所以这里需要3个持久卷.如果计划调度更多的副本,则需要创建更多的持久卷..

之前在学习存储知识的时候介绍过存储卷,所以具体不演示,以下是创建3个PV持久卷的配置文件

```yaml
[root@k8s-master ~]# cat statefulset-kubia-pv.yaml
apiVersion: v1
#创建一个List列表资源,List的items下列出每个PV的配置
kind: List
items:
  - apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv-1
    spec:
       capacity:
         storage: 1Gi
       accessModes:
         - ReadWriteOnce
       persistentVolumeReclaimPolicy: Recycle
       storageClassName: nfs
       nfs:
         path: /data/k8s/pv-1
         server: 172.16.20.1

  - apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv-2
    spec:
       capacity:
         storage: 1Gi
       accessModes:
         - ReadWriteOnce
       persistentVolumeReclaimPolicy: Recycle
       storageClassName: nfs
       nfs:
         path: /data/k8s/pv-2
         server: 172.16.20.1

  - apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv-3
    spec:
       capacity:
         storage: 1Gi
       accessModes:
         - ReadWriteOnce
       persistentVolumeReclaimPolicy: Recycle
       storageClassName: nfs
       nfs:
         path: /data/k8s/pv-3
         server: 172.16.20.1
```

> 以前接触过在yaml文件中添加---3个横杠使的在一个文件中可以区分定义多个资源,这次定义一个List对象,然后把各个资源作为List对象的各个项目.这2种方法均可以在一个YAML文件中定义多个资源

现在已经定义了个3个底层的PV持久卷

```
[root@k8s-master ~]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
mypv1   1Gi        RWO,ROX        Recycle          Available           nfs                     12d
pv-1    1Gi        RWO            Recycle          Available           nfs                     4s
pv-2    1Gi        RWO            Recycle          Available           nfs                     4s
pv-3    1Gi        RWO            Recycle          Available           nfs                     4s
```

---

### 创建Headless Service

下面是headless service的配置文件,唯一需要注意的是该类型服务的clusterIP属性必须为None

```yaml
apiVersion: v1
kind: Service
metadata:
  name: statefulset-kubia-svc
spec:
  clusterIP: None
  #所有标签为statefulset-kubia的Pod都属于这个Service
  selector:
    app: statefulset-kubia
  ports:
    - name: http
      port: 80
```

创建服务

```
[root@k8s-master ~]# kubectl get svc
NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
statefulset-kubia-svc   ClusterIP      None            <none>        80/TCP         3s
```

---

### 创建Statefuleset

​      statefulset资源的配置和RS,deployment等没有太大的区别,这里使用了一个新的组件volumeClaimTemplates.其中定义了一个持久卷声明.该组件会为每个Pod创建一个独立的持久卷声明.

​      这个组件是在statefulset资源的spec全局对象下,虽然在pod的template模板中并没有创建持久卷声明(而是直接通过volumeMounts属性来挂在).但是Statefulset在创建时,会自动将volumeClaimTemplate定义的持久卷声明关联到pod中.

``` yaml
[root@k8s-master ~]# cat statefulset-kubia.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: statefulset-kubia-v1
spec:
  serviceName: statefulset-kubia-v1
  replicas: 3
  selector:
    matchLabels:
       app: statefulset-kubia

  template:
    metadata:
      labels:
        app: statefulset-kubia

    spec:
      containers:
      - name: statefulset-kubia
        image: luksa/kubia-pet
        ports:
          - name: http
            containerPort: 8080
        volumeMounts:
            - name: data
              mountPath: /var/data

  volumeClaimTemplates:
       - metadata:
           name: data
         spec:
           resources:
              requests:
                 storage: 1Gi
           storageClassName: nfs
           accessModes:
             - ReadWriteOnce
```

> 注意:volumeClaimTemplates组件一定要声明存储类型storageClassName,如果没有声明这一点则Pod一直处于Pending状态.并且会有以下报错信息

```
[root@k8s-master ~]# kubectl describe po statefulset-kubia-v1-0

  Warning  FailedScheduling  <unknown>  default-scheduler  error while running "VolumeBinding" filter plugin for pod "statefulset-kubia-v1-0": pod has unbound immediate PersistentVolumeClaims
```

查看PVC提示没有找到PV

```
[root@k8s-master ~]# kubectl describe pvc data-statefulset-kubia-v1-0
Events:
  Type    Reason         Age                  From                         Message
  ----    ------         ----                 ----                         -------
  Normal  FailedBinding  55s (x182 over 45m)  persistentvolume-controller  no persistent volumes available for this claim and no storage class is set
```



   创建statefulset资源,列出pod资源.和rs,rc,deployment不同的是,他们会一次性创建完所有的pod,而statefulset会在每一个pod完全就绪后,才会创建第二个.

​    statefulset这样做是因为:状态明确的集群应用对同事有2个集群成员启动引起的竞争情况是非常敏感的.所以依次启动每个成员是比较安全可靠的.

```yaml
[root@k8s-master ~]# kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
statefulset-kubia-v1-0   0/1     Pending   0          74s
```

---

现在3个Pod副本都已经被创建完成.

```shell
[root@k8s-master ~]# kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
statefulset-kubia-v1-0   1/1     Running   0          12m
statefulset-kubia-v1-1   1/1     Running   0          12m
statefulset-kubia-v1-2   1/1     Running   0          12m
```



statefulset自动创建了3个PVC,并且各自与3个pv自动关联

```bash
[root@k8s-master ~]# kubectl get pvc
NAME                          STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-statefulset-kubia-v1-0   Bound    pv-2     1Gi        RWO            nfs            11m
data-statefulset-kubia-v1-1   Bound    pv-3     1Gi        RWO            nfs            11m
data-statefulset-kubia-v1-2   Bound    pv-1     1Gi        RWO            nfs            11m


[root@k8s-master ~]# kubectl get pv
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                 STORAGECLASS   REASON   AGE
pv-1   1Gi        RWO            Recycle          Bound    default/data-statefulset-kubia-v1-2   nfs                     5h18m
pv-2   1Gi        RWO            Recycle          Bound    default/data-statefulset-kubia-v1-0   nfs                     5h18m
pv-3   1Gi        RWO            Recycle          Bound    default/data-statefulset-kubia-v1-1   nfs                     5h18m
[root@k8s-master ~]#
```

​       和RS,RC,Deployment等资源不同的是,Statefulset部署的pod名称并非是随机的,而是pod模板名加上一个序号,这个序号从0开始,依次增加.

​       PVC的名称格式是PVC的名字+pod名.每个pod自动创建一个PVC,并且该PVC自动关联到一个后端的PV持久卷

---

### 访问POD

​     由于创建的Service类型是Headless service模式,所以不能通过它来访问pod,而是需要直接连接到每个后端单独的pod.(或者是创建一个普通的Service,但是这样也不允许访问指定的pod)

​    这次介绍如何通过API服务器与pod通信.API服务器可以通过代理直接连接到指定的pod.可以通过如下的URL

```
<apiServerHost>:<port>/api/v1/namespaces/default/pods/pods名称/proxy/<path>
```

 在k8s的master节点运行下面命令,下面命令运行一个kubectl proxy.从而可以让proxy去API服务器通信,而不必使用麻烦的授权和SSL证书来直接与API服务器通信

```
kubectl proxy
```

现在就可以直接访问Pod了.开启另一个master服务器终端.通过curl访问某个Pod.比如访问statefulset-kubia-v1-0这个Pod容器

```
[root@k8s-master ~]# curl localhost:8001/api/v1/namespaces/default/pods/statefulset-kubia-v1-0/proxy/
You've hit statefulset-kubia-v1-0
Data stored on this pod: No data posted yet
```

这种访问方式经过了2层的中间代理:

##### 1.curl命令发送给kubectl proxy

##### 2. kubectl proxy 带上认证TOKEN转发给API服务器

##### 3. API服务器再通过pod容器的实际IP地址将请求转发到后端的Pod

下面是发送一个post请求到statefulset-kubia-v1-0的例子

```
[root@k8s-master ~]# curl -X POST -d "Hey There ! This greeting was submitted to statefulset-kubia-v1-0" \
> localhost:8001/api/v1/namespaces/default/pods/statefulset-kubia-v1-0/proxy/
Data stored on pod statefulset-kubia-v1-0

#再次用GET请求,就可以返回刚才POST提交的数据
[root@k8s-master ~]# curl localhost:8001/api/v1/namespaces/default/pods/statefulset-kubia-v1-0/proxy/
You've hit statefulset-kubia-v1-0
Data stored on this pod: Hey There ! This greeting was submitted to statefulset-kubia-v1-0
```

当我们访问其他的pod容器时,并没有返回写入的数据,这和期望的一致,说明每个节点都有各自独立的存储状态

```
[root@k8s-master ~]# curl localhost:8001/api/v1/namespaces/default/pods/statefulset-kubia-v1-1/proxy/
You've hit statefulset-kubia-v1-1
Data stored on this pod: No data posted yet
```

---

### 删除pod,重新调度

之前我们在statefulset-kubia-v1-0这个pod节点写入了一条数据,这次我们删除这个Pod,等它被重新调度,然后检查它是否还会返回与之前一致的数据

```
[root@k8s-master ~]# kubectl delete po statefulset-kubia-v1-0
pod "statefulset-kubia-v1-0" deleted

[root@k8s-master ~]# kubectl get po
NAME                     READY   STATUS    RESTARTS   AGE
statefulset-kubia-v1-0   1/1     Running   0          2m
statefulset-kubia-v1-1   1/1     Running   0          17h
statefulset-kubia-v1-2   1/1     Running   0          17h

[root@k8s-master ~]# curl localhost:8001/api/v1/namespaces/default/pods/statefulset-kubia-v1-0/proxy/
You've hit statefulset-kubia-v1-0
Data stored on this pod: Hey There ! This greeting was submitted to statefulset-kubia-v1-0
[root@k8s-master ~]#
```

> 删除一个Pod,当Pod重新被调度时不一定是原节点,有可能会调度到另外一个节点

从上面的实验中可以得出2个结论:

* statefulset的pod被重新调度时,会新创建一个和之前一模一样的Pod(包括主机名称,pod名,存储)
* 当pod被删除,重新调度后持久化数据与之前一模一样.

---

### statefulSet滚动更新

#### 1.7版本之前默认的On Delete更新策略

​     statefulset在1.7版本开始支持滚动更新..在1.7版本之前默认的更新测量是`On Delete`.这种侧列和ReplicaSet类似.当更新了配置文件后,旧的pod并不会被自动删除,而是需要手动删除.

​    下面这个例子中,将镜像更换为luksa/kubia-pet-peers.副本数从3个增加到4个.(为此,我们需要提前再创建一个pv-4

```shell
#编辑pv配置文件,增加pv-4(前提是nfs服务器上实现存在/data/k8s/pv-4目录
[root@k8s-master ~]# vim statefulset-kubia-pv.yaml

#更新pv配置文件
[root@k8s-master ~]# kubectl apply -f statefulset-kubia-pv.yaml
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
persistentvolume/pv-1 configured
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
persistentvolume/pv-2 configured
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
persistentvolume/pv-3 configured
persistentvolume/pv-4 created

#查看PV.
[root@k8s-master ~]# kubectl get pv
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                 STORAGECLASS   REASON   AGE
pv-1   1Gi        RWO            Recycle          Bound       default/data-statefulset-kubia-v1-2   nfs                     23h
pv-2   1Gi        RWO            Recycle          Bound       default/data-statefulset-kubia-v1-0   nfs                     23h
pv-3   1Gi        RWO            Recycle          Bound       default/data-statefulset-kubia-v1-1   nfs                     23h
pv-4   1Gi        RWO            Recycle          Available                                         nfs                     9s
[root@k8s-master ~]#
```

更新statefulset配置文件

```
#更新配置文件
[root@k8s-master ~]# vim statefulset-kubia.yaml
#应用配置文件
[root@k8s-master ~]# kubectl apply -f statefulset-kubia.yaml
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
statefulset.apps/statefulset-kubia-v1 configured

#查看Pod
[root@k8s-master ~]# kubectl get pods
NAME                     READY   STATUS              RESTARTS   AGE
statefulset-kubia-v1-0   1/1     Running             0          70m
statefulset-kubia-v1-1   1/1     Running             0          18h
statefulset-kubia-v1-2   1/1     Running             0          18h
statefulset-kubia-v1-3   0/1     ContainerCreating   0          5s
[root@k8s-master ~]#
```

通过Pod的存活字段可以看到之前旧版本的Pod并没有被自动删除,而是新增了一个副本.这和ReplicaSet的机制类似.

----

#### 自动滚动更新策略

编辑statefulset配置文件,将镜像版本改回到luksa/kubia-pet

```yaml
spec:
  serviceName: statefulset-kubia-v1
  replicas: 4
  selector:
    matchLabels:
       app: statefulset-kubia
  #在spec字段配置更新策略,默认的type是On Delete,修改为RollingUpdate
  updateStrategy:
     type: RollingUpdate
```

应用新的配置文件,此时会触发自动更新

```bash
[root@k8s-master ~]# kubectl apply -f statefulset-kubia.yaml
statefulset.apps/statefulset-kubia-v1 configured
[root@k8s-master ~]# kubectl get pods
NAME                     READY   STATUS        RESTARTS   AGE
statefulset-kubia-v1-0   1/1     Running       0          5m22s
statefulset-kubia-v1-1   1/1     Running       0          6m12s
statefulset-kubia-v1-2   1/1     Running       0          6m57s
statefulset-kubia-v1-3   1/1     Terminating   0          7m39s

[root@k8s-master ~]# kubectl get pods
NAME                     READY   STATUS        RESTARTS   AGE
statefulset-kubia-v1-0   1/1     Running       0          6m55s
statefulset-kubia-v1-1   1/1     Terminating   0          7m45s
statefulset-kubia-v1-2   1/1     Running       0          14s
statefulset-kubia-v1-3   1/1     Running       0          54s

[root@k8s-master ~]# kubectl get pods
NAME                     READY   STATUS        RESTARTS   AGE
statefulset-kubia-v1-0   1/1     Terminating   0          7m27s
statefulset-kubia-v1-1   1/1     Running       0          7s
statefulset-kubia-v1-2   1/1     Running       0          46s
statefulset-kubia-v1-3   1/1     Running       0          86s
```

发现了什么? 当滚动更新时,kubectl会以倒序的方式,从最末尾一个pod开始依次更新.

> StatefulSet的滚动更新策略不同于Deployment可以指定maxSuge参数指定一次同时更新的pod数量,而是只能单个方式进行依次更新

> StatefulSet还支持partition(分区)的更新策略,具体可以查看官网

无论是何种更新策略.Pod的数据(包括主机名,存储)都会持久化.再次访问第0个pod,存储数据依然存在

```
[root@k8s-master ~]# curl localhost:8001/api/v1/namespaces/default/pods/statefulset-kubia-v1-0/proxy/
You've hit statefulset-kubia-v1-0
Data stored on this pod: Hey There ! This greeting was submitted to statefulset-kubia-v1-0
```

---

### StatefulSet 如何处理节点失效

在node2上关闭网卡来模拟这台服务器掉线,观察statefulSet处理节点失效的情况

> 注意关闭节点网卡前请确保可以通过控制台连接服务器,因为这意味着无法ssh远程登录

node2节点已经关闭,状态为notready

```
[root@k8s-master ~]# kubectl get node
NAME         STATUS     ROLES    AGE   VERSION
k8s-master   Ready      master   49d   v1.17.3
k8s-node1    Ready      <none>   49d   v1.17.3
k8s-node2    NotReady   <none>   49d   v1.17.3
```

过一段时间后,所有node2节点上的Pod为Terminating终止状态

```bash
[root@k8s-master ~]# kubectl get pods -o wide
NAME                     READY   STATUS        RESTARTS   AGE   IP               NODE        NOMINATED NODE   READINESS GATES
statefulset-kubia-v1-0   1/1     Terminating   0          33m   10.100.169.174   k8s-node2   <none>           <none>
statefulset-kubia-v1-1   1/1     Terminating   0          34m   10.100.169.173   k8s-node2   <none>           <none>
statefulset-kubia-v1-2   1/1     Running       0          34m   10.100.36.97     k8s-node1   <none>           <none>
statefulset-kubia-v1-3   1/1     Running       0          35m   10.100.36.99     k8s-node1   <none>           <none>
```

##### 删除不健康的Pod

当尝试手动删除pod时,发现永远都无法删除

```
[root@k8s-master ~]# kubectl delete po statefulset-kubia-v1-0
pod "statefulset-kubia-v1-0" deleted
^@
^@

```

在另一个终端上查看该pod.发现虽然pod被Terminating挂起,但是容器仍然处于运行状态

```bash
[root@k8s-master ~]# kubectl describe pods statefulset-kubia-v1-0
Name:                      statefulset-kubia-v1-0
Namespace:                 default
Priority:                  0
Node:                      k8s-node2/172.16.20.253
Start Time:                Sun, 03 May 2020 10:54:17 +0800
Labels:                    app=statefulset-kubia
                           controller-revision-hash=statefulset-kubia-v1-74b44bc68b
                           statefulset.kubernetes.io/pod-name=statefulset-kubia-v1-0
Annotations:               cni.projectcalico.org/podIP: 10.100.169.174/32
Status:                    Terminating (lasts 12m)
Termination Grace Period:  30s
IP:                        10.100.169.174
IPs:
  IP:           10.100.169.174
Controlled By:  StatefulSet/statefulset-kubia-v1
Containers:
  statefulset-kubia:
    Container ID:   docker://099628b95ded3644752a3de799ef338794704aaa4ebe4db5a966b821b2e9a71a
    Image:          luksa/kubia-pet
    Image ID:       docker-pullable://luksa/kubia-pet@sha256:4263bc375d3ae2f73fe7486818cab64c07f9cd4a645a7c71a07c1365a6e1a4d2
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sun, 03 May 2020 10:54:21 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/data from data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jfrqr (ro)
```

##### 强制删除

带上参数`--force --grace-period 0`可以强制删除一个pod

```
[root@k8s-master ~]# kubectl delete po statefulset-kubia-v1-0 --force --grace-period 0
warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "statefulset-kubia-v1-0" force deleted

[root@k8s-master ~]# kubectl get pods
NAME                     READY   STATUS              RESTARTS   AGE
statefulset-kubia-v1-0   0/1     ContainerCreating   0          5s
statefulset-kubia-v1-1   1/1     Terminating         0          44m
statefulset-kubia-v1-2   1/1     Running             0          44m
statefulset-kubia-v1-3   1/1     Running             0          45m

[root@k8s-master ~]# kubectl get pods -o wide
NAME                     READY   STATUS        RESTARTS   AGE   IP               NODE        NOMINATED NODE   READINESS GATES
statefulset-kubia-v1-0   1/1     Running       0          12s   10.100.36.101    k8s-node1   <none>           <none>
statefulset-kubia-v1-1   1/1     Terminating   0          44m   10.100.169.173   k8s-node2   <none>           <none>
statefulset-kubia-v1-2   1/1     Running       0          44m   10.100.36.97     k8s-node1   <none>           <none>
statefulset-kubia-v1-3   1/1     Running       0          45m   10.100.36.99     k8s-node1   <none>           <none>
```

此时第0个pod已经重新创建,并且运行到node1节点,而另一个pod依然处于Terminating状态

##### node2节点恢复正常

当节点恢复正常后,很快node和pod都全部恢复正常.此时第0个pod依然还是挂在在node1节点.第1个pod的状态迅速从Terminating状态变为Running状态

```
[root@k8s-master ~]# kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP               NODE        NOMINATED NODE   READINESS GATES
statefulset-kubia-v1-0   1/1     Running   0          3m17s   10.100.36.101    k8s-node1   <none>           <none>
statefulset-kubia-v1-1   1/1     Running   0          5s      10.100.169.172   k8s-node2   <none>           <none>
statefulset-kubia-v1-2   1/1     Running   0          47m     10.100.36.97     k8s-node1   <none>           <none>
statefulset-kubia-v1-3   1/1     Running   0          48m     10.100.36.99     k8s-node1   <none>           <none>
```

---

### 本章总结

Stateful和RS,deployment的用法总体没有太大区别,下面是这2种资源的对比

|       特性       |                         Deployment                         |                        StatefulSet                         |
| :--------------: | :--------------------------------------------------------: | :--------------------------------------------------------: |
|  是否暴露到外网  |                            可以                            |                           一般不                           |
|  请求面向的对象  |                        ServiceName                         |                       指定pod的域名                        |
|      灵活性      |             通过Service(名称或者IP)访问后端Pod             |                    可以访问任意一个pod                     |
|      易用性      |                 只需要关心Service信息即可                  |         需要知道访问pod的名称,Headless Service名称         |
|   PV/PVC稳定性   |                      无法保障绑定关系                      |                          可以保障                          |
|  pod名称稳定性   | 使用一个随机的名称后缀,重启后会随机生成另外一个.名称不重复 |                      稳定,每次都一样                       |
|   升级更新顺序   | 随机启动.如果pod宕机重启,也是随机分配一个Node节点重新启动  | pod按顺序依次启动,如果pod宕机,依然使用相同的Node节点和名称 |
|     停止顺序     |                          随机停止                          |                          倒序停止                          |
| 集群内部服务发现 |              只能通过Service访问随机的一个Pod              |                   可以打通pod之间的通信                    |
|     性能开销     |                无需维护pod与node,pvc等关系                 |                   需要维护额外的关系信息                   |

通过对比发现

* 如果不需要额外数据依赖或者状态维护的部署,优先选择Deployment
* 如果单纯要做数据持久化,方式pod宕机数据丢失,直接使用PV/PVC就可以
* 如果是有多个副本,且每个副本挂载的PV存储数据不同,并且pod宕机重启后仍然关联到之前的PVC,并且数据需要持久化,考虑使用StatefulSet

