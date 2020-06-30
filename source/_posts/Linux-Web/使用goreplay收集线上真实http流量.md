---
title: 使用goreplay收集线上真实http流量
date: 2020-06-29 11:59:58
tags: goreplay
categories: Linux-Web
comments: true
copyright: true
---



## 使用goreplay收集线上真实http流量

### 背景介绍

在很多场景中,我们需要将线上服务器的真实Http请求复制转发到某台服务器中(或者测试环境中),并且前提是不影响线上生产业务进行.

例如:

1. 通常可能会通过ab等压测工具来对单一http接口进行压测。但如果是需要http服务整体压测，使用ab来压测工作量大且不方便，通过线上流量复制引流，通过将真实请求流量放大N倍来进行压测，能对服务有一个较为全面的检验.
2. 将线上流量引入到测试环境中,测试某个中间件或者数据库的压力
3. 上线前在预发布环境，使用线上真实的请求，检查是否准备发布的版本，是否具备发布标准
4. 用线上的流量转发到预发，检查相同流量下一些指标的反馈情况，检查核心数据是否有改善、优化.

<!--more-->

---

### goreplay介绍

goreplay项目请参考github:[goreplay介绍](https://github.com/buger/goreplay)

goreplay是一款开源网络监控工具,可以在不影响业务的情况下,记录服务器真实流量,将该流量用来做镜像,压力测试,监控和分析等用途.

简单来说就是goreplay抓取线上真实的流量，并将捕捉到的流量转发到测试服务器上(或者保存到本地文件中)

goreplay大致工作流程如下:

![](https://img2.jesse.top/20200629110035.png)

---

### goreplay常见使用方式

goreplay使用文档参考:[goreplay文档](https://github.com/buger/goreplay/wiki)

常用的一些命令

```
-input-raw 抓取指定端口的流量 gor --input-raw :8080
-output-stdout 打印到控制台
-output-file 将请求写到文件中 gor --input-raw :80 --output-file ./requests.gor
-input-file 从文件中读取请求，与上一条命令呼应 gor --input-file ./requests.gor
-exit-after 5s 持续时间
-http-allow-url url白名单，其他请求将会被丢弃
-http-allow-method 根据请求方式过滤
-http-disallow-url 遇上一个url相反，黑名单，其他的请求会被捕获到
```

> 更多命令可以使用 ./gor --help查看

---

### goreplay安装

在github上下载Linux的二进制文件: [goreplay安装](https://github.com/buger/goreplay/releases)

> 注意.虽然在github上提供了rpm安装包,但是实际安装发现无法安装:

```
[root@dwd-tongji-3 ~]# rpm -ivh gor-1.0.0-1.x86_64.rpm
Preparing...                          ################################# [100%]
	package goreplay-1.0.0-1.x86_64 is intended for a different operating system
```

下载github上的二进制文件,解压后是一个gor的二进制可执行文件.复制到PATH变量路径下即可

```
[root@dwd-tongji-3 ~]# wget https://github.com/buger/goreplay/releases/download/v1.0.0/gor_1.0.0_x64.tar.gz
[root@dwd-tongji-3 ~]# ls
 gor_1.0.0_x64.tar.gz
[root@dwd-tongji-3 ~]# tar -xf gor_1.0.0_x64.tar.gz
[root@dwd-tongji-3 ~]# ls
gor 
[root@dwd-tongji-3 ~]# ll gor
-rwxr-xr-x 1 501 games 17779040 Mar 30  2019 gor
[root@dwd-tongji-3 ~]# cp gor /usr/local/bin/
```

---

### goreplay简单实践

#### 1.将本地http的流量保存到本地文件中.

为了简便起见,以下命令都在root用户下执行

```
## 1.开启一个screen窗口
[root@dwd-tongji-3 ~]# screen -S GOR
## 2.将80流量保存到本地的文件
[root@dwd-tongji-3 ~]# gor --input-raw :80 --output-file /data/requests.gor
Version: 1.0.0
```

默认情况下goreplay会以块文件存储,将流量保存为多个块文件

```
[root@dwd-tongji-3 ~]# ls /data/requests_* | more
/data/requests_0.gor
/data/requests_100.gor
/data/requests_101.gor
/data/requests_102.gor
/data/requests_103.gor
/data/requests_104.gor
/data/requests_105.gor
/data/requests_106.gor
/data/requests_107.gor
/data/requests_108.gor
/data/requests_109.gor
/data/requests_10.gor
```

使用`--output-file-append`选项可以将捕获流量保存为一个单独的文件

```
[root@dwd-tongji-3 ~]#./gor --input-raw :80 --output-file /data/gor.gor --output-file-append

[root@dwd-tongji-3 ~]# ll /data -h
total 1.4M
-rw-r----- 1 root root 1.4M Jun 29 15:13 gor.gor
```

#### 2.将http的请求打印到终端

```
[root@dwd-tongji-3 ~]#gor --input-raw :8000 --output-stdout
```

#### 3.将http的请求转发到测试环境

```
gor --input-raw :80 --output-http="http://beta:80"
```

在测试服务器上的nginx查看日志,.发现流量已经进来了

```
10.111.51.243 - - [29/Jun/2020:14:56:55 +0800] "POST /piwik.php HTTP/1.1" 200 5 "https://2021001151691008.hybrid.alipay-eco.com/2021001151691008/0.2.2006111453.18/index.html#pages/index/index?appid=2021001151691008&taskId=415" "Mozilla/5.0 (iPhone; CPU iPhone OS 13_3_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/17D50 Ariver/1.0.12 AliApp(AP/10.1.95.7030) Nebula WK RVKType(0) AlipayDefined(nt:4G,ws:375|667|2.0) AlipayClient/10.1.95.7030 Language/zh-Hans Region/CN NebulaX/1.0.0" "112.96.179.238"
10.111.51.243 - - [29/Jun/2020:14:56:55 +0800] "POST /piwik.php HTTP/1.1" 200 5 "https://servicewechat.com/wxa090d3923fde0d4b/132/page-frame.html" "Mozilla/5.0 (iPhone; CPU iPhone OS 13_5_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148 MicroMessenger/7.0.13(0x17000d29) NetType/4G Language/zh_CN" "14.106.171.11"
10.111.51.243 - - [29/Jun/2020:14:56:55 +0800] "POST /piwik_new.php?actionname=zt-template HTTP/1.1" 400 249 "-" "-" "118.31.36.251"
10.111.51.243 - - [29/Jun/2020:14:56:55 +0800] "POST /piwik_new.php?actionname=zt-template HTTP/1.1" 400 249 "-" "-" "118.31.36.251"
10.111.51.243 - - [29/Jun/2020:14:56:55 +0800] "POST /piwik_new.php?actionname=zt-template HTTP/1.1" 400 249 "-" "-" "118.31.36.251"
```

####  也可以将流量输出到多个终端

* 输出到多个http服务器

```
gor --input-tcp :28020 --output-http "http://staging.com"  --output-http "http://dev.com"
```

* 输出到文件或者Http服务器

```
gor --input-raw :80 --output-file requests.log --output-http "http://staging.com"
```

####  4.将流量从文件重放到http服务器

1.首先将请求流量保存到本地文件

```
sudo ./gor --input-raw :8000 --output-file=requests.gor
```

2.再开一个窗口,运行gor,将请求流量从文件中重放

```
./gor --input-file requests.gor --output-http="http://localhost:8001"
```

---

### 压力测试

goreplay支持将捕获到的生产实际请求流量减少或者放大重播以用于测试环境的压力测试.压力测试一般针对Input流量减少或者放大.例如下面的例子

```
# Replay from file on 2x speed
#将请求流量以2倍的速度放大重播
gor --input-file "requests.gor|200%" --output-http "staging.com"
```

当然也也支持10%,20%等缩小请求流量

---

### 限速

如果受限于测试环境的服务器资源压力,只想重播一部分流量到测试环境中,而不需要所有的实际生产流量,那么就可以用限速功能.有两种策略可以实现限流

1.随机丢弃请求流量

2.基于Header或者URL丢弃一定的流量(百分比)

#####  随机丢弃请求流量

input和output两端都支持限速,有两种限速算法:**百分比**或者**绝对值**

* 百分比: input端支持缩小或者放大请求流量,基于指定的策略随机丢弃请求流量
* 绝对值: 如果单位时间(秒)内达到临界值,则丢弃剩余请求流量,下一秒临界值还原

**用法**:

在output终端使用"|"运算符指定限速阈值,例如:

* 使用绝对值限速

```
# staging.server will not get more than ten requests per second
#staging服务每秒只接收10个请求
gor --input-tcp :28020 --output-http "http://staging.com|10"
```

* 使用百分比限速

```
# replay server will not get more than 10% of requests 
# useful for high-load environments
gor --input-raw :80 --output-tcp "replay.local:28020|10%"
```

##### 基于Header或者URL参数限速

如果header或者URL参数中有唯一值,例如(API key),则可以转发指定百分比的流量到后端,例如:

```
# Limit based on header value
gor --input-raw :80 --output-tcp "replay.local:28020|10%" --http-header-limiter "X-API-KEY: 10%"

# Limit based on URL param value
gor --input-raw :80 --output-tcp "replay.local:28020|10%" --http-param-limiter "api_key: 10%"
```

---

###  过滤

如果只想捕获指定的URL路径请求,或者http头部,或者Http方法,则可以使用过滤功能

下面是几个例子

* 只捕获某个URL

```
# only forward requests being sent to the /api endpoint
gor --input-raw :8080 --output-http staging.com --http-allow-url /api
```

* 拒绝某个URL

```
# only forward requests NOT being sent to the /api... endpoint
gor --input-raw :8080 --output-http staging.com --http-disallow-url /api
```

* 基于正则表达式过滤头部

```
# only forward requests with an api version of 1.0x
gor --input-raw :8080 --output-http staging.com --http-allow-header api-version:^1\.0\d

# only forward requests NOT containing User-Agent header value "Replayed by Gor"
gor --input-raw :8080 --output-http staging.com --http-disallow-header "User-Agent: Replayed by Gor"
```

* 过滤HTTP请求方法

```
gor --input-raw :80 --output-http "http://staging.server" \
    --http-allow-method GET \
    --http-allow-method OPTIONS
```

