---
title: kubernetes volume
date: 2020-06-26 11:59:58
tags: k8s
categories: kubernetes
comments: true
copyright: true
---



## kubernetes volume

## 介绍

在学习docker时,学习了如何将宿主机上的文件系统挂载到容器当中,实现持久化存储,以及多个容器共享同一个文件目录.

在k8s中,我们也希望pod容器中的数据能够持久化存储.当一个容器因为故障或者其他原因需要删除时,我们希望新的容器能够在上一个容器结束的位置继续运行.

但是和docker一样,容器默认情况下删除后,所有该容器产生的数据都会消失.并且,虽然容器和宿主机共享CPU,网络,内存等资源,但是并不会共享文件存储.甚至同一个Pod中每个容器都有自己独立的文件系统,彼此互相隔离.

和docker容器一样,这就需要有一种存储卷能够挂载到Pod容器中,并脱离Pod的生命周期之外,将容器运行产生的数据保存在宿主机或者独立的外部存储卷中

<!--more-->

---

## 存储卷类型

* emptyDir-------用于存储容器临时数据的空目录

emptyDir类型的存储卷和pod容器的生命周期相关联当pod容器删除时,emtpyDir卷的数据就会丢失

* hostPath--------和docker类似,将宿主机的某个目录挂载到pod容器.

但是本地节点运行的Pod容器和其他节点服务器上的同一组pod容器之间无法共享数据.只能本地持久化,无法实现集群存储

* nfs,iscsi,FC SAN-------网络存储

网络存储设备,挂载一个独立的第三方存储设备

* AWS EBK,Azure Disk等------ 云厂商虚拟存储

* cephfs,glusterfs等----- 分布式集群存储

> 使用```kubectl explain pods.spec.volumes```命令可以查看k8s支持的许多存储卷类型

----

## 一. emptyDir卷

为了演示emtpyDir卷,现在创建2个简单的pod容器:

nginx-alpine容器.

fortune容器定期输出字符串到/var/hotdocs/index.html文件下.然后Nginx容器展示.

这2个容器需要共享同一个目录.


```
#furtune容器镜像主要是运行一个while循环脚本,该脚本每隔10秒随机想/var/htdocs/index.html文件写入一段名人名言

#!/bin/bash
trap ”exit” SIGINT
mkdir /var/htdocs 
while :
do
echo $(date) Writing fortune to /var/htdocs/index.html 
/usr/games/fortune > /var/htdocs/index.html
sleep 10
done


然后编译Dockerfile的文件，其中包含以下内容:

FROM ubuntu:latest
RUN apt-get update;apt-get -y install fortune 
ADD fortuneloop.sh /bin/fortuneloop.sh
ENTRYPOINT /bin/fortuneloop.sh

```
使用```kubectl explain pod.volumes```命令可以查看k8s支持哪些存储卷类型


在pod定义的sepc.containers.volumeMounts字段中详细介绍了挂载磁盘卷的配置参数.

使用命令 ``` kubectl explian pod.spec.containers.volumeMounts```可以看到支持以下主要挂载参数

```
[root@k8s-master ~]# kubectl explain pod.spec.containers.volumeMounts
KIND:     Pod 
VERSION:  v1  #pod对象属于哪个版本

#volumeMounts参数的值是一个列表对象
RESOURCE: volumeMounts <[]Object>

FIELDS:
#挂载路径.必要字段
   mountPath	<string> -required-
     Path within the container at which the volume should be mounted. Must not
     contain ':'.

#挂载的volume卷名,必要字段
   name	<string> -required-
     This must match the Name of a Volume.

#是否只读
   readOnly	<boolean>
     Mounted read-only if true, read-write otherwise (false or unspecified).
     Defaults to false.

```

为了演示实验是否正常,同时复习之前学到的replicaSet和Service对象,我们来创建一个rs,和svc对象

####  1. 创建replicaSet对象.包含一个副本,以及上面配置的2个pod容器

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
    name: fortune-rs

spec:
   replicas: 1
   selector:
       matchLabels:
           app: fortune

   template:
      metadata:
         name: fortune
         labels:
           app: fortune
      spec:
         containers:
             - image: luksa/fortune
               name: html-generator
               #磁盘卷挂载配置
               volumeMounts:
                    - name: html  #卷名
                      mountPath: /var/htdocs #挂载在容器的哪个目录下

             - image: nginx:alpine
               name: web-servier
               volumeMounts:
                   - name: html
                     mountPath: /usr/share/nginx/html
                     readOnly: true
               ports:
                   - name: http
                     containerPort: 80
         #磁盘卷定义
         volumes:
         #卷名
            - name: html
              #卷类型
              emptyDir: {}
```

pod包含2个容器,他们挂载同一个公用的存储卷,当html-generator容器启动时,每10秒启动一次fortune命令输出到/var/htdocs/index.html文件中,因为卷同时也被web-server容器挂载,所以后者能访问到html-generator容器生成的数据

如果要将emptyDir卷存储在内存上而非磁盘,可以声明medium配置如下配置:

```
 volumes:
         #卷名
            - name: html
              #卷类型
              emptyDir: 
                 #类型
                 medium: Memory

```

#### 2.为此ReplicaSet创建一个Service对象.内容如下

```

apiVersion: v1
kind: Service
metadata:
   name: fortune-svc

spec:
   selector:
     app: fortune

   ports:
     - port: 80
       targetPort: 80
       
       
```

#### 3.分别创建svc和rs对象

```
kubectl create -f fortune-rs.yaml

kubectl create -f fortune-svc.yaml
```

#### 4.查看资源

```
[root@k8s-master ~]# kubectl get rs
NAME         DESIRED   CURRENT   READY   AGE
fortune-rs   1         1         1       11h

[root@k8s-master ~]# kubectl get svc
NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
fortune-svc          ClusterIP      10.96.123.110   <none>        80/TCP         7m9s

[root@k8s-master ~]# kubectl get pods
NAME                READY   STATUS    RESTARTS   AGE
fortune-rs-4df5q    2/2     Running   0          11m
```

#### 5.访问Service

```
[work@k8s-node1 ~]$ curl http://10.96.123.110
You never have to change anything you got up in the middle of the night
to write.
		-- Saul Bellow
[work@k8s-node1 ~]$ curl http://10.96.123.110
Q:	How much does it cost to ride the Unibus?
A:	2 bits.
[work@k8s-node1 ~]$ curl http://10.96.123.110
While you recently had your problems on the run, they've regrouped and
are making another attack.

```
可以看到,每10秒钟访问的内容不一样

---

## 二 .hostPath卷

hostPath卷可以实现持久性存储,如果删除了一个Pod,并且新的pod使用了相同的主机路径的hostPath卷,则新pod会发现上一个pod留下来的数据,但是前提是必须和前一个pod是同一个节点

这也解释了为什么使用hostPath卷不是一个好主意,因为当Pod被调度到另外一个节点上时,会找不到数据.


## 三.NFS卷

NFS卷可以实现多个节点之间共享存储.但是生产中不建议这样做,因为NFS共享存储传输效率低,稳定性和安全性不高.

仍然使用上面的例子

```
[root@k8s-master ~]# cp fortune-svc.yaml fortune-nfs-svc.yaml
[root@k8s-master ~]# cp fortune-rs.yaml fortune-nfs-rs.yaml
```
修改各资源的标签,和rs配置文件中的卷信息:

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
    name: fortune-nfs-rs

spec:
   replicas: 1
   selector:
       matchLabels:
           app: fortune-nfs

   template:
      metadata:
         name: fortune-nfs
         labels:
           app: fortune-nfs
      spec:
         containers:
             - image: luksa/fortune
               name: html-generator
               #磁盘卷挂载配置
               volumeMounts:
                    - name: html  #卷名
                      mountPath: /var/htdocs #挂载在容器的哪个目录下

             - image: nginx:alpine
               name: web-servier
               volumeMounts:
                   - name: html
                     mountPath: /usr/share/nginx/html
                     readOnly: true
               ports:
                   - name: http
                     containerPort: 80
         #磁盘卷定义
         volumes:
         #卷名
            - name: html
              #卷类型
              nfs:
            #NFS服务器地址
               server: 172.16.20.2
            #NFS服务端共享目录
               path: /data/apps/k8s

```

启动rs后,在nfs服务器上已经看到pod写入的数据:

```
[work@idc-beta-cron ~]$ cat /data/apps/k8s/index.html
Tonight's the night: Sleep in a eucalyptus tree.

[work@idc-beta-cron ~]$ cat /data/apps/k8s/index.html
Fortune: You will be attacked next Wednesday at 3:15 p.m. by six samurai
sword wielding purple fish glued to Harley-Davidson motorcycles.

Oh, and have a nice day!
		-- Bryce Nesbitt '84


```
---

## PV和PVC

PV和PVC用于kubenetes将存储和pod解耦,存储管理人员配置好PV,然后开发人员只需要声明需要多大的存储卷(pvc)即可,无需关系底层的存储实现方式.

**实现逻辑大致如下:**

1.存储管理员配置底层存储方案(NFS,SAN,FC SAN,ceph等)

2.k8s集群管理员创建一个PV(Psersistent volume,持久卷)

3.开发人员(或者用户)创建一个PVC(Persistent volume claim,持久卷声明),将PVC和PV绑定

4.开发人员(或者用户)在pod中引用PVC

下面用NFS底层存储来演示PV和PVC的工作过程

#### 1.创建PV

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mypv1

spec:
   capacity:
      storage: 1Gi   #定义PV卷的大小

   accessModes:
      - ReadWriteOnce   #PV可以读写模式被挂载到单个节点
      - ReadOnlyMany    #PV以只读模式被挂载到多个节点

   persistentVolumeReclaimPolicy: Retain  #PV的回收策略.Retain表示PVC释放后,PV会继续保留
   storageClassName: nfs   #指定nfs类型
   nfs:
     path: /data/k8s
     server: 172.16.20.1
```

* accessModes支持ReadWriteOnce ,ReadOnlyMany ,ReadWriteMany 

可以直接使用RWO,ROM,RWM缩写.

* PersistentVolumeReclaimPolicy回收策略支持:
  - Retain:  PV一直保留,直到管理员手动回收
  - Recycle: 清除PV中的数据
  - Delete:  清除存储上的资源

* storageClassName: PV的类型


```
[root@k8s-master ~]# kubectl create -f nfs-pv.yaml
persistentvolume/mypv1 created
[root@k8s-master ~]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
mypv1   1Gi        RWO,ROX        Retain           Available           nfs                     5s
[root@k8s-master ~]#

```

#### 2.持久卷声明(PVC)

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc1

spec:
  accessModes: #访问模式
     - ReadWriteOnce
  resources: #定义需要的存储空间大小
     requests:
       storage: 1Gi
  storageClassName: nfs #存储类型
```

创建PVC

```
[root@k8s-master ~]# kubectl create -f nfs-pvc.yaml
persistentvolumeclaim/mypvc1 created
[root@k8s-master ~]# kubectl get pvc
NAME     STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mypvc1   Bound    mypv1    1Gi        RWO,ROX        nfs            10s
[root@k8s-master ~]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   REASON   AGE
mypv1   1Gi        RWO,ROX        Retain           Bound    default/mypvc1   nfs                     22m
[root@k8s-master ~]#
```

default/mypvc1 表示default命名空间的PVC

Status的Bound表示PV和PVC已经成功绑定

> 持久卷PV不属于任何名称空间,但是PVC和pod有名称空间概念.PV可以被所有名称空间下的PVC绑定.


#### 3.创建pod.在pod中调用刚才创建的PVC

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
    name: fortune-nfs-pvc-rs

spec:
   replicas: 1
   selector:
       matchLabels:
           app: fortune-pvc-nfs

   template:
      metadata:
         name: fortune-pvc-nfs
         labels:
           app: fortune-pvc-nfs
      spec:
         containers:
             - image: luksa/fortune
               name: html-generator
               #磁盘卷挂载配置
               volumeMounts:
                    - name: html  #卷名
                      mountPath: /var/htdocs #挂载在容器的哪个目录下

             - image: nginx:alpine
               name: web-servier
               volumeMounts:
                   - name: html
                     mountPath: /usr/share/nginx/html
                     readOnly: true
               ports:
                   - name: http
                     containerPort: 80
         #磁盘卷定义
         volumes:
         #卷名
            - name: html
              #卷类型
              persistentVolumeClaim:
                 claimName: mypvc1
```

和empty dir以及Hostpath类似,在引用pvc的时候只是修改一下卷类型.引用mypvc1这个刚才创建的pvc

创建pod

```
[root@k8s-master ~]# kubectl create -f fortune-nfs-pvc.yaml
replicaset.apps/fortune-nfs-pvc-rs created

[root@k8s-master ~]# kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
fortune-nfs-pvc-rs-zlnk2   2/2     Running   0          26s
kubia-5452q                0/1     Running   0          5d19h
kubia-6mghh                0/1     Running   0          5d19h
kubia-nl6rd                0/1     Running   0          5d19h
kubia-pvs5z                0/1     Running   0          5d19h
ssd-monitor-5vxbn          1/1     Running   0          5d19h
```
#### 4.验证

在nfs服务器上的共享目录/data/k8s上确实看到了Pod容器写入的数据

```
[work@idc-beta-docker ~]$ ll /data/k8s/
total 4
-rw-r--r-- 1 root root 276 Apr 19 10:24 index.html
[work@idc-beta-docker ~]$ cat /data/k8s/index.html
Mind!  I don't mean to say that I know, of my own knowledge, what there is
particularly dead about a door-nail.  I might have been inclined, myself,
to regard a coffin-nail as the deadest piece of ironmongery in the trade.
But the wisdom of our ancestors is in the simile; and my unhallowed hands
shall not disturb it, or the Country's done for.  You will therefore permit
me to repeat, emphatically, that Marley was as dead as a door-nail.
		-- Charles Dickens, "A Christmas Carol"
[work@idc-beta-docker ~]$

```

-------

## pv的手动回收

手动删除Pod,pvc

```
[root@k8s-master ~]# kubectl delete pvc mypvc1
persistentvolumeclaim "mypvc1" deleted

[root@k8s-master ~]# kubectl get pvc
No resources found in default namespace.

[root@k8s-master ~]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM            STORAGECLASS   REASON   AGE
mypv1   1Gi        RWO,ROX        Retain           Released   default/mypvc1   nfs                     11h
[root@k8s-master ~]#
```

删除pvc后,pv是Released状态,不可绑定PVC.删除PV,然后重新创建.

```
[root@k8s-master ~]# kubectl delete pv mypv1
persistentvolume "mypv1" deleted

[work@idc-beta-docker ~]$ ll /data/k8s/
total 4
-rw-r--r-- 1 root root 100 Apr 19 10:38 index.html

```
删除PV卷,并不会删除存储卷中的数据

重新创建PV.存储卷中的数据仍然存在,并且PV的状态为Available

```

[root@k8s-master ~]# kubectl create -f nfs-pv.yaml
persistentvolume/mypv1 created

[root@k8s-master ~]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
mypv1   1Gi        RWO,ROX        Retain           Available           nfs                     6s

[work@idc-beta-docker ~]$ ll /data/k8s/
total 4
-rw-r--r-- 1 root root 100 Apr 19 10:38 index.html
[work@idc-beta-docker ~]$

```
---

## PV自动回收

在pv的配置中修改如下字段

```
persistentVolumeReclaimPolicy: Recycle
```

创建PV.Reclaim policy是Recycle

```

[root@k8s-master ~]# kubectl create -f nfs-pv.yaml
persistentvolume/mypv1 created
[root@k8s-master ~]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
mypv1   1Gi        RWO,ROX        Recycle          Available           nfs                     3s
[root@k8s-master ~]#

```

创建PVC,绑定到PV,并且创建pod

```
[root@k8s-master ~]# kubectl create -f nfs-pvc.yaml
persistentvolumeclaim/mypvc1 created

[root@k8s-master ~]# kubectl create -f fortune-nfs-pvc.yaml
replicaset.apps/fortune-nfs-pvc-rs created

[root@k8s-master ~]# kubectl get pvc
NAME     STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mypvc1   Bound    mypv1    1Gi        RWO,ROX        nfs            26s

[root@k8s-master ~]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   REASON   AGE
mypv1   1Gi        RWO,ROX        Recycle          Bound    default/mypvc1   nfs                     101s

[root@k8s-master ~]# kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
fortune-nfs-pvc-rs-9p2xz   2/2     Running   0          16s

```

nfs底层存储卷上数据已经更新 

```
[work@idc-beta-docker ~]$ cat /data/k8s/index.html
Alimony and bribes will engage a large share of your wealth.
		
```

#### 删除Pod,pvc

```
[root@k8s-master ~]# kubectl delete pvc mypvc1
persistentvolumeclaim "mypvc1" deleted

[root@k8s-master ~]# kubectl get pvc
No resources found in default namespace.
```

再次查看PV发现状态又变成了Available

```
[root@k8s-master ~]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM            STORAGECLASS   REASON   AGE
mypv1   1Gi        RWO,ROX        Recycle          Released   default/mypvc1   nfs                     5m19s
[root@k8s-master ~]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
mypv1   1Gi        RWO,ROX        Recycle          Available           nfs                     5m21s
```

> NFS不支持delete回收策略,所以就不演示
