---
title:  记一次生产ELK性能优化
date: 2020-08-25 22:59:58
tags:  elk
categories: elk
comments: true
copyright: true
---

### 记一次生产ELK性能优化

ES上线后遇到一些问题:

#####  一.内存压力过高

目前ES节点的有两种规格内存.

80GB内存(节点同时运行了Logstash,分配了16G给logstash).

54GB内存(只运行ES,并且只作为data节点)

ES的JVM内存从32G下降到28G.建议JVM内存是总物理内存的一半左右,

节点内存使用情况如下:

<!--more-->

```
"mem": { - 
          "total": "78.5gb",
          "total_in_bytes": 84296601600,
          "free": "4.3gb",
          "free_in_bytes": 4688633856,
          "used": "74.1gb",
          "used_in_bytes": 79607967744,
          "free_percent": 6,
          "used_percent": 94
```

建议: ES节点内存建议配置高一点,虽然ES的JVM内存最大不超过32G.但是在其他方面仍然很吃内存

---

##### 二.索引字段数

ES的单个索引默认最大字段数量是1000个.当超过这个字段数量时,会拒绝字段映射.

```
[2020-08-14T10:44:10,605][INFO ][o.e.a.b.TransportShardBulkAction] [idc-function-elk01] [logstash-mg-tc-netrcd-gateway-2020.08.14][0] mapping update rejected by primary
java.lang.IllegalArgumentException: Limit of total fields [10000] in index [logstash-mg-tc-netrcd-gateway-2020.08.14] has been exceeded
```

此时可以在索引模板中,修改默认字段数量.

```
PUT _template/logstash
{
    "index_patterns": "logstash-*",
    "settings": {
        "mapping.total_fields.limit":5000
     }
}
```

---

##### 三.ES线程和队列优化

ES日志有部分警告信息:

```
[2020-08-14T10:15:06,151][WARN ][o.e.c.r.a.AllocationService] [idc-function-elk01] failing shard [failed shard, shard [logstash-msf-internal-access-2020.08.14][0], node[3uwnN_KEQGiywi
u9xR5Jng], [R], s[STARTED], a[id=ACE7jGGoS6yhS9T_Sn53UA], message [failed to perform indices:data/write/bulk[s] on replica [logstash-msf-internal-access-2020.08.14][0], node[3uwnN_KEQ
Giywiu9xR5Jng], [R], s[STARTED], a[id=ACE7jGGoS6yhS9T_Sn53UA]], failure [RemoteTransportException[[idc-function-elk08][172.16.20.108:9300][indices:data/write/bulk[s][r]]]; nested: Cir
cuitBreakingException[[parent] Data too large, data for [<transport_request>] would be [20720204958/19.2gb], which is larger than the limit of [20401094656/19gb], real usage: [2068074
9096/19.2gb], new bytes reserved: [39455862/37.6mb], usages [request=197056/192.4kb, fielddata=1310/1.2kb, in_flight_requests=11463397212/10.6gb, accounting=54576960/52mb]]; ], markAs
Stale [true]]
org.elasticsearch.transport.RemoteTransportException: [idc-function-elk08][172.16.20.108:9300][indices:data/write/bulk[s][r]]
Caused by: org.elasticsearch.common.breaker.CircuitBreakingException: [parent] Data too large, data for [<transport_request>] would be [20720204958/19.2gb], which is larger than the l
imit of [20401094656/19gb], real usage: [20680749096/19.2gb], new bytes reserved: [39455862/37.6mb], usages [request=197056/192.4kb, fielddata=1310/1.2kb, in_flight_requests=114633972
12/10.6gb, accounting=54576960/52mb]
        at org.elasticsearch.indices.breaker.HierarchyCircuitBreakerService.checkParentLimit(HierarchyCircuitBreakerService.java:347) ~[elasticsearch-7.7.0.jar:7.7.0]
        at org.elasticsearch.common.breaker.ChildMemoryCircuitBreaker.addEstimateBytesAndMaybeBreak(ChildMemoryCircuitBreaker.java:128) ~[elasticsearch-7.7.0.jar:7.7.0]
```

编辑`/etc/elasticsearch/elasticsearch.yml`配置文件.新增如下配置:

```
thread_pool:
    write:
        size: 32
        queue_size: 10000
processors: 32
```

同时,修改`/etc/logstash/logstash.yml`配置文件.修改如下配置

```
pipeline.batch.size: 10000
pipeline.batch.delay: 100
```

> 以上这些配置都需要不断的调试,修改,观察,找到最适合的一个范围和效果

---



##### 四.集群最大分片(shard)数

今天收集新的日志报错,logstash日志内容如下

```
[2020-08-24T10:32:41,009][WARN ][logstash.outputs.elasticsearch][main][7ca4981ae091d6b8f604e5695405b2c74a9cb7fc4a72b338be4e2ede66a04d7d] Could not index event to Elasticsearch. {:status=>400, :action=>["index", {:_id=>nil, :_index=>"logstash-hsq-search-netrcd-2020.08.24", :routing=>nil, :_type=>"_doc"}, #<LogStash::Event:0x4a0b491>], :response=>{"index"=>{"_index"=>"logstash-hsq-search-netrcd-2020.08.24", "_type"=>"_doc", "_id"=>nil, "status"=>400, "error"=>{"type"=>"illegal_argument_exception", "reason"=>"Validation Failed: 1: this action would add [14] total shards, but this cluster currently has [8998]/[9000] maximum shards open;"}}}}
```

看日志关键字发现当前集群总共分片数是8998个,如果再加上14个分片,那么就超过了最大的9000个.

**14和9000个分片是怎么来的?**

当前ES集群中有7个热节点,每个索引分片数量是7个,副本数是1..也就是说一个索引就要14个分片(包括副本)

另外,集群中还有2个冷节点,总共是9个节点.从Elasticsearch v7.0.0 开始，集群中的每个节点默认限制 1000 个shard.所以集群所有节点总共是9000个shard(分片)

**解决方案**

1.每个索引(Index)分配多少个分片(shared)合适?

配置Elasticsearch集群后,对于分片的数量通常比较难确定.分配过小或者过大对性能都不好.实际上每个分片都会消耗硬件资源:

- 由于分片本质上是Lucene索引，因此会消耗文件句柄，内存和CPU资源。
- 每个搜索请求都将触摸索引中每个分片的副本，当分片分布在多个节点上时，这不是问题。当分片争夺相同的硬件资源时，就会出现争用并且性能会下降。

我个人理解一般设置分片数量有几个方向可以考虑

* 由于Elasticsearch的最大JVM一般在30-32G.所以一个分片的数量不能超过30G大小.如果一个索引最大是200G.那么就需要分片7个分片.
* 最好是按照日志创建索引,可以按天,或者周,甚至月来创建索引,如果一个月的日志太大,那么就按天创建索引
* 分片数量的配置最好是基于当前的日志量级,集群节点数量评估.或者可以规划为可以预见的增长幅度评估,千万不能盲目的为一年或者2年以后的日志量分配资源
* 如果不能确定索引容量,或者对于分片设置完全没有概念和把握,那么建议按照ES集群节点数量设置分片,有多少个节点,就设置多少分片,后期再观察和考虑是否需要增加或删减
* 生产环境中,最好设置为1个副本.副本有助于加速查询和健壮高可用行,但是也会占用磁盘容量空间,没必要设置2个或以上的副本.如果是做了冷热分离,对于冷数据如果没有太大的安全性要求,也可以不设置副本

 2.每个节点的`maximum shards open`设置为多大合适

**根据以下几个指标来评估:**

1.当前集群的shard分片数

2.当前集群索引量

3.索引保存时间

以我们生产集群为例,当前集群共有7个热节点,每个索引7个分片.每节点是一个分片.每天是55个索引,加上副本是110个索引,也就是每个节点,每天有110个shard分片.

热节点索引保留14天,单节点总共是110*14=1540个Shard.预留20%的空间,也就是1848个分片.所以给每个节点分配1800-2000个分片比较合适

另外,集群中还有2个冷节点.超过14天的索引会自动迁移到热节点(没有副本),并且保留14天后删除.所以冷节点14天总共分片数是55\*7\*14=5390个Shard.分摊到2个节点,平均每个节点的Shard是2695..留出20%的空间,每节点的Shard为3234.

**经过评估下来,ES热节点每节点Shard数量2000,冷节点3500**

编辑ES配置文件,添加以下配置,修改每节点的Shard数量

* 热节点: `cluster.max_shards_per_node: 2000`

* 冷节点: `cluster.max_shards_per_node: 3500`

以下是Ansible发布模板

```
{% if es_role  is defined and es_role == 'hot' %}
cluster.max_shards_per_node: 2000
{% elif es_role  is defined and es_role == 'cold' %}
cluster.max_shards_per_node: 3500
{% endif %}
```

重启ES后并没有生效.临时在Kibana的dev tool中设置:

```
PUT /_cluster/settings
{
  "transient": {
    "cluster": {
      "max_shards_per_node":3000
    }
  }
}
```

设置立即生效,在ELK集群面板中,总的shard分片数超过了9000的最大值,已经达到9,376 shards,

博客参考:https://studygolang.com/articles/25396

---

