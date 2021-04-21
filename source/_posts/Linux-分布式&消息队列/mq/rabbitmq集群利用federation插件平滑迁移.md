---
title: rabbitmq集群利用federation插件平滑迁移
date: 2018-11-21 17:59:58
tags: MQ
categories: [Linux-分布式&消息队列,mq ]
comments: true
copyright: true
---

## rabbitmq集群利用federation插件平滑迁移

关于feration的官方文档:[federated Exchange](<https://www.rabbitmq.com/federated-exchanges.html>)

关于安装和配置feration的文档:[RabbitMQ Federation集群](<https://www.jianshu.com/p/5a8b00cf4a0a>)



### 背景

公司的MQ集群出现了点问题,需要将业务迁移到一台新的MQ集群(包含2个MQ节点服务器).但是由于生产环境中一直有生产和消费数据产生,而且更麻烦的是还有大量延迟队列(生产的数据在一天甚至一周后再去消费).

此时如何做到不影响业务,不影响数据一致性,也不产生2次重复消费的前提下平滑迁移就很有挑战性.直接暴力的将业务迁移到新的rabbitmq集群显然不行.

此时就要求新集群服务器和老集群数据同步,并且将业务迁移到新的集群后,老的集群上未消费的数据,或者延迟队列的数据仍然在迁移后仍然能被消费掉..

federation插件完美的完成了这个任务

---

<!--more-->

### 介绍

federation插件可以将源MQ服务器,甚至源MQ集群(又称上游服务器/集群)的消息实时复制到另外一台MQ服务器节点或者MQ集群(又称下游服务器/集群)

上游并不需要和下游并不需要在同一个节点或者同一个集群,甚至不需要在同一个网络中..rabbitmq的版本.erlang的版本也不要求一致性.

上游和下游同时遵循amqp通信协议.只需要网络能连接,就可以实现同步.

这个过程有点类似于mysql的主从同步.但是federation插件更简单,因为几乎不需要对上游进行任何变更或者操作.只需要在下游服务器安装配置federation插件即可.而安装过程又是如此的简单

---



### 安装

安装方法很简单,在rabbitmq 节点服务器上直接启用federation插件即可

**启动rabbitmq_federation插件**

```
root@hsq-mq-node1-temp:~# rabbitmq-plugins enable rabbitmq_federation
The following plugins have been enabled:
  rabbitmq_federation

Applying plugin configuration to rabbit@hsq-mq-node1-temp... started 1 plugin.
```

**启用rabbitmq_feration_management插件**

```
root@hsq-mq-node1-temp:~# rabbitmq-plugins enable rabbitmq_federation_management
The following plugins have been enabled:
  rabbitmq_federation_management

Applying plugin configuration to rabbit@hsq-mq-node1-temp... started 1 plugin.
```

启用插件后就可以在rabbitmq的web控制台的"Admin"菜单中看到Federation status和Federation upstreams菜单

---

### 同步配置

1. **定义策略**

>  federation同步插件配置可以参考笔记开头中的文档

在刚刚安装了federation插件的下游rabbitmq管理平台上定义一个policies策略:

```
#策略如下:
Name: 自定义一个名称
Pattern: 这里是匹配一个队列或者交换机..可以使用^符合代表匹配所有
Apply to: 选择Excahnges and queues表示应用到交换机和队列
Priority: 不用填写
Definition: federation-upstream-set:	all  #表示从所有上游节点同步
```



**2.定义上游节点**

在"Federation Upstreams"菜单中,点击"Add a new upstream" .然后进行如下配置

```
name" 自定义
URI: 上游节点的路径.amqp://dwd:WhgOPshbfw@10.111.30.201  #前面表示rabbitmq的账号密码,后面是rabbitmq集群的IP地址
Expires: 360000  #上游节点缓存消息时间.单位毫秒
```

定义完成后,就可以在Federation Status菜单中看到插件的"State"为running状态

---

#### 论证

经过测试,发现向上游rabbitmq生产和消费数据,会自动同步到下游rabbitmq集群.主要进行了下列测试

**测试一:** 

上游集群和下游集群同时生产数据..下游时候会被上游数据覆盖

**结论:**

2个集群同时在同一个队列生产数据.下游的数据会一直累积,是上游数据的2倍..并不会被上游集群数据覆盖

---

**测试二:**

上游集群生产,但是不消费.下游集群消费

**结论:**

下游会一直消费上游集群同步过来的数据

---



**测试三:**

上游集群生产,并且消费.是否会同步到下游

**结论:**

会同步到下游,但是下游集群没有消费者去消费,所以数据一直积压

---

**测试四:**

上游rabbitmq的版本不变,仍然是3.6.3 .但是将下游rabbitmq的版本升级到3.6.5.插件是否仍然正常工作

**结论:**

版本不一致,不影响插件正常运行

---

**测试五:**

上游rabbitmq持续生产..Consumers到下游rabbitmq去消费.

**结论:**

上游rabbitmq和下游rabbitmq同时被消费.consumers实际上消费了2份同样的数据

---

### 平滑迁移方案

结合上述同步的工作特点.平滑迁移步骤如下:

1. 新的rabbitmq集群所有节点安装federation插件.

2. 新的rabbitmq集群导入一份线上的元数据(包括Exchange,queue等状态信息)

3. 将业务服务器的consumer消费者连接地址切换到新的rabbitmq集群

4. 配置federation插件.(这样可以一直同步上游rabbitmq生产的数据.但是可能会消费2份同样的数据)

5. 将业务服务器的Publish生产者连接地址切换到新的rabbitmq集群

6. 观察业务,rabbitmq生产消费,以及延迟消息队列数据的情况

7. 保持老rabbitmq集群正常运行.因为可能还有一些延迟队列插件的消息没有被消费或者同步

8. 经过观察和确认,业务运行稳定后,在新的rabbitmq集群的web管理控制台停止和删除federation同步策略以及配置

9. 在新的rabbitmq集群节点关闭federation插件

   

   