---
title: ES集群优化
date: 2019-01-7 09:59:58
tags: elasticsearch
categories: [elasticsearch ]
comments: true
copyright: true
---


### ES集群优化



#### 背景

时间:2019年01月3号

支付宝五福活动压测期间

---

#### ES集群架构

elasticsearch版本:2.4.6

ES集群服务器: 8台.其中5台16c32g.3台8c16g

服务器节点: mq-master,mq-slave,hsq-es1,hsq-es2,hsq-es3,hsq-es4,hsq-es5,hsq-es6

<!--more-->

---

#### 资料参考

官方资料:[thread pool](https://www.elastic.co/guide/en/elasticsearch/reference/2.4/modules-threadpool.html)

官方资料:[系统优化](https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html)

---

#### 优化步骤

* **系统内核优化**

**最大文件打开数**设置为65535还是有点小.

```
[work@hsq-es1 elasticsearch]$ grep "65536" hsq_elasticsearch.log.2019-01-02
[2019-01-02 14:17:14,740][WARN ][env                      ] [hsq-es1] max file descriptors [65535] for elasticsearch process likely too low, consider increasing to at least [65536]
```

修改为102400.修改方式如下:

1.编辑 /etc/security/limits.conf:

```
* soft noproc  65535
* hard noproc 65535
* soft nofile 102400
* hard nofile 102400
```

然后退出,重新登录shell

2.如果是supervisor方式启动的进程.还需要修改supervisord.conf文件:

```
#修改下面一行
minfds=102400                 ; (min. avail startup file descriptors;default 1024)
```



* **虚拟内存**定义了进程能拥有的最多内存区域.这个是ES官方文档推荐的配置

```
[work@hsq-es1 elasticsearch]$ tail -3 /etc/sysctl.conf
fs.file-max=102400

vm.max_map_count = 262144  #这一行配置
```



* **memlock**最大锁定内存地址空间 

> memlock这步优化实际中没做,当时没将pam_limits.so文件加入启动文件中

在 /etc/security/limits.conf中加入如下内容

```
* soft memlock unlimited
* hard memlock unlimited
```

> 要使limits.conf文件配置生效，必须要确保pam_limits.so文件被加入到启动文件中。

确保/etc/pam.d/login文件中有如下内容:

```
session required /lib/security/pam_limits.so
```

验证是否生效

```
curl localhost:9200/_nodes/stats/process?pretty
```

---

* **内存优化**

由于新加入的es服务器节点都是16c32g的配置,内存分配总物理内存的70%.大概是22G左右.

**1.编辑环境变量**

```
vim /etc/profile

#定义如下环境变量
export ES_HEAP_SIZE=11g

source /etc/profile
```



2. 如果是supervisor方式启动的进程,还需要定义supervisor配置文件

```
[work@hsq-es1 elasticsearch]$ vim /etc/supervisord/conf.d/elasticsearch.conf

#加入如下一行:
environment=ES_HEAP_SIZE=22g
environment=MAX_OPEN_FILES=102400
```

通过ps命令可以看到elasticsearch进程的最大内存数

```
[work@hsq-es2 ~]$ ps aux | grep elasticsearch
work       782 68.6 62.1 50121636 20372716 ?   Sl   Jan03 925:22 /bin/java -Xms24g -Xmx24g -Djava.awt.headless=true -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:+HeapDumpOnOutOfMemoryError -XX:+DisableExplicitGC -Dfile.encoding=UTF-8 -Djna.nosys=true -Des.path.home=/data/app/elasticsearch-2.4.6 -cp /data/app/elasticsearch-2.4.6/lib/elasticsearch-2.4.6.jar:/data/app/elasticsearch-2.4.6/lib/* org.elasticsearch.bootstrap.Elasticsearch start
```

---



* **elasticsearch程序优化**

编辑elasticsearch.conf配置文件

```
#线程池的配置###
#fixed 索引线程池类型
threadpool.index.type: fixed
#线程池大小,建议等同于CPU核心数
threadpool.index.size: 16
#队列大小
threadpool.index.queue_size: 6000

#搜索 线程池类型

#搜索线程池大小,建议2倍的CPU核心数
threadpool.search.size: 32
# 搜索线程池类型
threadpool.search.type: fixed
#队列大小
threadpool.search.queue_size: 6000

processors: 16

# # 缓存类型设置为Soft Reference，只有当内存不够时才会进行回收
index.cache.field.max_size: 50000
index.cache.field.expire: 10m
index.cache.field.type: soft
#
# 查询缓存
indices.queries.cache.size: 20%
index.queries.cache.enabled: true

indices.requests.cache.size: 10%
index.requests.cache.enable: true

indices.fielddata.cache.size: 20%

# #适当增大写入buffer和bulk队列长度
indices.memory.index_buffer_size: 15%
thread_pool.bulk.queue_size: 1024
```

---

* **es集群节点优化**

参考官方文档: [Node](https://www.elastic.co/guide/en/elasticsearch/reference/2.4/modules-node.html)

关于ES的节点介绍:

Elasticsearch集群的每个elasticsearch实例都是一个node节点.多个node节点组成了一个cluster.

集群的每个节点默认都能处理http和传输层流量.传输层专门用于节点之间以及节点和Java TransportClient之间的通信; HTTP层仅由外部REST客户端使用。

所有节点都可以转发客户端请求到合适的其他节点,除此之外,每个节点服务器还可以承担如下角色:

* Master node

默认情况下,每个节点都是Master.master节点不代表这个节点就是master角色,而是代表这个节点有参加master选举的资格.master节点控制整个集群

通过如下方式设置为仅为master节点:

```
node.master: true
node.data: false
node.client: false
```

master节点不会存储数据，有成为主节点的资格，可以参与选举，有可能成为真正的主节点。普通服务器即可(CPU、内存消耗一般)。



* Data node

默认情况下,每个节点都是Data节点,Date节点保存数据,并且执行与数据相关的操作,例如搜索,聚合.

节点没有成为主节点的资格，不参与选举，只会存储数据。在集群中需要单独设置几个这样的节点负责存储数据，后期提供存储和查询服务。主要消耗磁盘，内存。

通过如下方式设置为仅为数据节点:

```
node.master: false
node.data: true
node.client: false
```



* Client node

client节点不会保存数据,也不会成为master角色,client节点用来转发客户端的请求到master节点, 转发数据相关的操作请求(例如搜索)到data节点,

通过如下方式设置为仅为client节点

```
node.master: false
node.data: false
```

不会成为主节点，也不会存储数据，主要是针对海量请求的时候可以进行负载均衡。普通服务器即可（如果要进行分组聚合操作的话，建议这个节点内存也分配多一点）

---

* **节点规划**

```
master 节点 （管理集群作用）

hsq-es2
hsq-es2-mqslave


data 数据节点 数据落地查询

hsq-es2 该节点也作为master节点 
hsq-es3
hsq-es4
hsq-es5
hsq-es6


client节点（不作为master节点 也不作为data 节点） 负载和聚合

hsq-es2-mqmaster
hsq-es1
```

附上hsq-es6的配置

```
vim elasticsearch.yml

node.name: hsq-es6
node.master: false
node.data: true
node.client: false
```

---

* **主分片优化**

访问http://es.haoshiqi.net/_plugin/kopf/#!/nodes 可以看到集群所有节点

![](/Users/huangyong/Desktop/es-1.png)

其中可以看到有2台master节点,2台client节点,4台data节点



将主分片平均分担到各个date节点,如下所示

![](/Users/huangyong/Desktop/es-2.png)

高亮显示的为主分片



调整主分片方法如下:

1.左键点击高亮主分片.点击"select for relocation"

![](/Users/huangyong/Desktop/es-3.png)

2.点击需要调整后的位置

![](/Users/huangyong/Desktop/es-4.png)

3.调整后结果如下

![](/Users/huangyong/Desktop/es-5.png)

---