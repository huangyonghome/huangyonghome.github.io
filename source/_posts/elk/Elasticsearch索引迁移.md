---
title:  Elasticsearch索引迁移	
date: 2020-09-30 22:59:58
tags:  elk
categories: elk
comments: true
copyright: true
---



## Elasticsearch索引迁移	

### 介绍

旧Elasticsearch版本:2.4.4

新Elasticsearch版本:2.4.4

近期dev环境服务器迁移到一台新的物理机,所以需要迁移部分Elasticsearch索引数据.

Elasticsearch索引迁移有许多方法.测试过elasticsearch-exporter.但是没有成功.报错如下:

```
[work@docker elasticsearch-exporter]$ node exporter.js -a 10.0.0.250 -b 10.0.0.101 -p 9200 -q 9200 -i mid_mg_gc_datasource_items -j mid_mg_gc_datasource_items
Elasticsearch Exporter - Version 1.4.0
Reading source statistics from ElasticSearch
The source driver has not reported any documents that can be exported. Not exporting.
Number of calls:	0
Fetched Entries:	0 documents
Processed Entries:	0 documents
Source DB Size:		0 documents
```

<!--more-->

### Elasticsearch-dump

本次使用elasticsearch-dump进行索引迁移.在github上可以找到具体使用方法:[elasticsearch-dump](https://github.com/elasticsearch-dump/elasticsearch-dump)

#### 安装

```
npm install elasticdump
```

这里稍微踩了个坑,如果报错:

```
pm WARN deprecated nomnom@1.8.1: Package no longer supported. Contact support@npmjs.com for more info.
npm WARN saveError ENOENT: no such file or directory, open '/home/work/package.json'
npm WARN enoent ENOENT: no such file or directory, open '/home/work/package.json'
npm WARN work No description
npm WARN work No repository field.
npm WARN work No README data
npm WARN work No license field.
```

则需要初始化一下npm

```
work@docker ~]$ npm init
```

安装完成后,进入到`elasticsearch dump`的`bin`目录下

```
[work@docker ~]$ cd node_modules/elasticdump/bin/
```

### 用法

查看elasticsearchdump的具体用法

[work@docker bin]$ ./elasticdump --help

`elaticsearchdump`支持两个ES跨版本迁移索引,还支持索引备份到文件,以及从文件恢复到Elasticsearch

#####  迁移mid_gm_gc_brand这个索引数据

```
[work@docker bin]$ ./elasticdump --input=http://10.0.0.250:9200/mid_mg_gc_brand --output=http://10.0.0.101:9200/mid_mg_gc_brand --type=analyzer

[work@docker bin]$ ./elasticdump --input=http://10.0.0.250:9200/mid_mg_gc_brand --

[work@docker bin]$ ./elasticdump --input=http://10.0.0.250:9200/mid_mg_gc_brand --output=http://10.0.0.101:9200/mid_mg_gc_brand --type=data
```

> 文档中的type类型有settings, analyzer, data, mapping, alias, template



##### 查看新服务器上的索引.迁移成功

```
 huangyong@huangyong-Macbook-Pro  ~  curl -Ssl 'http://10.0.0.101:9200/_cat/indices?v' | grep 'mid_mg*'
yellow open   mid_mg_gc_datasource_items             5   1         96            1     87.3kb         87.3kb
yellow open   mid_mg_gc_brand                        5   1        966            0    312.1kb        312.1kb
yellow open   mid_mg_gc_synonyms                     5   1        116            0     82.1kb         82.1kb

 huangyong@huangyong-Macbook-Pro  ~  curl -Ssl 'http://10.0.0.250:9200/_cat/indices?v' | grep 'mid_mg*'
yellow open   mid_mg_gc_datasource_items             3   1         92            9     99.1kb         99.1kb
yellow open   mid_mg_gc_brand                        3   1        966            0    345.4kb        345.4kb
yellow open   mid_mg_gc_synonyms                     3   1        116           16    106.1kb        106.1kb
```

