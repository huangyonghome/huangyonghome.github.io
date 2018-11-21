---
title: rabbitmq+haproxy 集群环境搭建
date: 2018-11-21 17:59:58
tags: MQ
categories: [Linux-分布式&消息队列 ]
comments: true
copyright: true
---

## 线上MQ宕机复盘

---

### 背景

时间:2018年11月20号晚上10点40

服务器:mq-slave

故障现象: 钉钉收到报警MQ服务器的rabbitmq进程挂了.手动启动后,过一会超时退出

---

1.查看日志.提示delayed_message插件超时.无法启动.

联系开发.可能是白天修改了延时队列机制的缘故

```
less /var/log/rabbitmq/rabbit@node2

Error: {{case_clause,{timeout,[rabbit_delayed_messagerabbit@node2]}}, [{rabbit_boot_steps,'-run_step/2-lc$^1/1-1-',1, [{file,"src/rabbit_boot_steps.erl"},{line,49}]}, {rabbit_boot_steps,run_step,2, [{file,"src/rabbit_boot_steps.erl"},{line,49}]}, {rabbit_boot_steps,'-run_boot_steps/1-lc$^0/1-0-',1, [{file,"src/rabbit_boot_steps.erl"},{line,26}]}, {rabbit_boot_steps,run_boot_steps,1, [{file,"src/rabbit_boot_steps.erl"},{line,26}]}, {rabbit,start_apps,1,[{file,"src/rabbit.erl"},{line,447}]}, {rabbit_plugins,ensure,1,[{file,"src/rabbit_plugins.erl"},{line,49}]}, {rpc,'-handle_call_call/6-fun-0-',5,[{file,"rpc.erl"},{line,206}]}]}
```

2.关闭该插件

```
root@node2:~# rabbitmq-plugins disable rabbitmq_delayed_message_exchange
he following plugins have been disabled: 
rabbitmq_delayed_message_exchange
```

3.再次启动.可以成功启动

```
root@node2:~# service rabbitmq-server start
```

4.尝试手动启动该插件.仍然超时失败

```
root@node2:~# rabbitmq-plugins enable rabbitmq_delayed_message_exchange

The following plugins have been enabled:
rabbitmq_delayed_message_exchange

Applying plugin configuration to rabbit@node2... failed. Error: {{case_clause,{timeout,[rabbit_delayed_messagerabbit@node2]}}, [{rabbit_boot_steps,'-run_step/2-lc$^1/1-1-',1, [{file,"src/rabbit_boot_steps.erl"},{line,49}]}, {rabbit_boot_steps,run_step,2, [{file,"src/rabbit_boot_steps.erl"},{line,49}]}, {rabbit_boot_steps,'-run_boot_steps/1-lc$^0/1-0-',1, [{file,"src/rabbit_boot_steps.erl"},{line,26}]}, {rabbit_boot_steps,run_boot_steps,1, [{file,"src/rabbit_boot_steps.erl"},{line,26}]}, {rabbit,start_apps,1,[{file,"src/rabbit.erl"},{line,447}]}, {rabbit_plugins,ensure,1,[{file,"src/rabbit_plugins.erl"},{line,49}]}, {rpc,'-handle_call_call/6-fun-0-',5,[{file,"rpc.erl"},{line,206}]}]}
```

5.停止rabbitmq服务

```
root@node2:~# service rabbitmq-server stop
```

---

联系开发.可能是延迟队列消息太多,将服务器的MQ程序卡死,导致插件无响应.从而无法启动.由于MQ使用了2台服务器座位集群,而且使用了镜像队列方式.所以清洗mq-slave服务器这台服务器的数据.重新启动.

确定是否使用镜像队列方式可以通过以下命令查看.

```
root@node2:~# rabbitmqctl list_policies
Listing policies ...
/	ha-all	all	^	{"ha-mode":"all","ha-sync-mode":"automatic"}	0
```

ha-mode: 代表使用镜像队列.

ha-sync-mode:表示自动同步数据

更多信息请网上搜索

---

1.关闭插件,重新启动rabbitmq进程.(因为清除数据需要先启动rabbitmq进程)

```
root@node2:~# service rabbitmq-server start
```

2.将这台服务器从集群节点拿掉.不然清楚数据会影响现有的生产环境

```
root@node2:~# rabbitmqctl stop_app
Stopping node rabbit@node2 ...
```

3.数据删除完毕后,重新启动rabbitmq节点

```
root@node2:~# rabbitmqctl start_app Starting node rabbit@node2 ...
```

4.启动插件.显示已经启动了

```
oot@node2:~# rabbitmq-plugins enable rabbitmq_delayed_message_exchange
Plugin configuration unchanged.
Applying plugin configuration to rabbit@node2... nothing to do.
```

5.显示插件,查看确实已经成功启动

```
root@node2:~# rabbitmq-plugins list -E

Configured: E = explicitly enabled; e = implicitly enabled
 | Status:   * = running on rabbit@node2
 |/
[E*] rabbitmq_delayed_message_exchange 0.0.1
[E*] rabbitmq_management               3.6.3
```

6.查看节点状态

```
rabbitmqctl status
```

7.查看集群状态.可以看到只识别到本身这台的节点,没有加入到集群

```
root@node2:~# rabbitmqctl cluster_status 

Cluster status of node rabbit@node2 ... 
[{nodes,[{disc,[rabbit@node2]}]}, 
{running_nodes,[rabbit@node2]},
{cluster_name,<<"rabbit@node2">>}, 
{partitions,[]}, {alarms,[{rabbit@node2,[]}]}]
```

8.关闭这台服务器节点

```
root@node2:~# rabbitmqctl stop_app Stopping node rabbit@node2 ...
```

9.加入另外一台node1的集群

```
root@node2:~# rabbitmqctl join_cluster rabbit@node1 
Clustering node rabbit@node2 with rabbit@node1 ...
```

10.启动节点

```
root@node2:~# rabbitmqctl start_app 
Starting node rabbit@node2 ...
```

11.查看集群状态.可以看到2个节点都识别到了

```
root@node2:~# rabbitmqctl cluster_status
Cluster status of node rabbit@node2 ...
[{nodes,[{disc,[rabbit@node1,rabbit@node2]}]},
 {running_nodes,[rabbit@node1,rabbit@node2]},
 {cluster_name,<<"rabbit@node1">>},
 {partitions,[]},
 {alarms,[{rabbit@node1,[]},{rabbit@node2,[]}]}]
root@node2:~#
```

12.查看日志,可以看到集群正在往该节点上同步数据

```
=WARNING REPORT==== 21-Nov-2018::00:39:47 === msg_store_persistent: rebuilding indices from scratch

=INFO REPORT==== 21-Nov-2018::00:39:48 === Mirrored queue 'hsq.marketingcenter.finish_pin_event' in vhost '/': Adding mirror on node rabbit@node2: <0.11440.0>

=INFO REPORT==== 21-Nov-2018::00:39:48 === Mirrored queue 'hsq.msgcenter.app_push_notification' in vhost '/': Adding mirror on node rabbit@node2: <0.11444.0>

=INFO REPORT==== 21-Nov-2018::00:39:48 === Mirrored queue 'hsq.msgcenter.lottery_push_notification' in vhost '/': Adding mirror on node rabbit@node2: <0.11448.0>

=INFO REPORT==== 21-Nov-2018::00:39:48 === Mirrored queue 'push_robot' in vhost '/': Adding mirror on node rabbit@node2: <0.11452.0>

=INFO REPORT==== 21-Nov-2018::00:39:48 === Mirrored queue 'hsq.tradecenter.refund_point' in vhost '/': Adding mirror on node rabbit@node2: <0.11456.0>
```

13.在本节点上查看队列.可以看到队列已经同步过来了

```
root@node2:~# rabbitmqctl list_queues
```

另外在web控制台上还能看到更详细的信息.

