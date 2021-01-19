---
title: Kong实现限流
date: 2021-01-19 11:59:58
tags:  kong
categories: [Linux-Web, kong]
comments: true
copyright: true
---

## Kong实现限流

### 介绍

近期发现公司某个业务对外的openapi接口的/merchantapi路径异常调用非常频繁.公司的第三方商户需要通过这个路径来调用ERP接口,但是经常发生被恶意刷接口的情况,导致公司的业务服务器资源使用率飙升,面临很大的宕机风险和隐患.

目前外部客户端访问公司业务仍然是阿里云SLB-----Nginx---php-fpm的架构.由于Nginx的限流能力并不出色,特别是针对具体path路径的限流.所以,引入了Kong api网关

### Rate Limiting限流插件介绍

Rate Limiting是Kong社区版就已经自带的官方流量控制插件.详细信息可以参考Kong官网介绍. https://docs.konghq.com/hub/kong-inc/rate-limiting/

它可以针对`consumer` ,`credential` ,`ip` ,`service`,`path`,`header` 等多种维度来进行限流.流量控制的精准度也有多种方式可以参考,比如可以做到秒级,分钟级,小时级等限流控制.

#### 响应客户端头部信息

当启用这个插件后.Kong会响应客户端一些额外的头部信息,告诉客户端限流信息.例如下面是Kong响应给客户端的header信息,告诉客户端当前的限流策略是10r/s

```
RateLimit-Limit: 10
RateLimit-Remaining: 0
RateLimit-Reset: 1

X-Kong-Response-Latency: 1
X-RateLimit-Limit-Second: 10
X-RateLimit-Remaining-Second: 0
```

如果客户端的访问请求超过限流的阈值,Kong会返回status`429`的状态码以及下面的错误信息

```
{ "message": "API rate limit exceeded" }
```

<!--more-->

#### 限流策略对Kong性能影响

Rate limiting插件支持3种限流策略.

`cluster` 集群策略.Kong的数据库会维护一个计数器,并且在所有的Kong集群内每个节点共享这个计数器.如果计数器触发限流上线,所有的Kong节点都拒绝客户端的转发.这就意味着每个节点接收到客户端的请求,都会对数据库进行读写操作.

`redis` redis策略和`cluster` 相似,唯一不同的是,计数器是存储在redis数据中.并且在集群内所有节点共享.

`local` 本地策略.计数器保存在Kong节点服务器本地内存缓冲区.并且计数器只对该节点有效.这意味着`local`策略有最好的性能表现.但是由于计数器存储在本地.所以限流的精度没有`redis`和`cluster` 准确.并且会影响Kong节点服务器弹性扩容(比如限流设置30r/s,Kong集群从2个节点扩容到4个节点.限流就从60r/s变成了120r/s.此时需要手动将限流设置从30r/s降低到15r/s)

> 或者,可以在Kong前面配置一个hash转发策略的负载均衡,将同一个外部客户端的请求代理到同一个节点.这样local策略的精确度可以提升,并且kong节点的弹性扩容不会影响限流效果

下面是3种限流策略的对比表

| policy  | describe  | pros                              | cons                                                |
| ------- | --------- | --------------------------------- | --------------------------------------------------- |
| cluster | 集群策略  | 限流精准度高,不需要第三方组件支持 | 对Kong性能影响比较大                                |
| redis   | redis策略 | 限流精准度高,对Kong性能影响较低   | 需要额外的redis服务                                 |
| local   | 本地策略  | 对Kong性能影响最低                | 精准度比较差,Kong节点扩容和缩容需要手动调整限流速率 |

下面是以上集群策略的使用场景:

* 如果对流量精确度要求非常高.比如金融,交易等.那么适合redis或者cluster的限流策略
* 如果是为了保护后端服务,避免大流量带来的服务器过载.那么适合local限流策略,这种场景对限流的精度要求不高



### 针对客户端IP限流

我们场景中针对客户端IP进行限流.但是由于Kong是在SLB或者Nginx的负载均衡后面,所以默认情况下,Kong采用的IP是上一级负载均衡器的IP.此时就需要将客户端的真实IP传递到Kong,并且使用该IP作为`remote_ip` .实现方法如下:

* 虚拟机运行Kong

针对rpm包或者其他方式安装的Kong服务,可以修改默认的`/etc/kong/kong.conf` 配置文件.加入下面2行配置信息:

```
trusted_ips = 0.0.0.0/0,::/0
real_ip_header = X-Forwarded-For
```

重载kong配置

```
kong reload
```

* docker容器方式运行kong

针对docker容器方式运行的Kong,修改配置文件不方便,此时可以通过变量注入的方式自定义配置`kong.conf` 配置文件.还可以通过这种方式注入nginx自定义配置,具体可以参考官方的文档介绍:[environment-variables](https://docs.konghq.com/2.2.x/configuration/#environment-variables)

例如,上面的2行配置内容可以通过在配置参数前面加`KONG_`以及大写的参数名的方式注入环境变量

```
KONG_TRUSTED_IPS=0.0.0.0/0,::/0
KONG_REAL_IP_HEADER=X-Forwarded-For
```

修改Kong的`docker-compose`文件:

```
kong:
    image: dwd-kong:2.2.0  #自定义kong镜像.也可以使用官方的docker镜像
    container_name: kong
    hostname: kong
    environment:
      - KONG_DATABASE=postgres
      - KONG_PG_HOST=kong-database
      - KONG_PROXY_ACCESS_LOG=/var/log/kong/access.log
      - KONG_ADMIN_ACCESS_LOG=/var/log/kong/admin_access.log
      - KONG_PROXY_ERROR_LOG=/var/log/kong/error.log
      - KONG_ADMIN_ERROR_LOG=/var/log/kong/admin_error.log
      - KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl
      - KONG_TRUSTED_IPS=0.0.0.0/0,::/0       #增加这两行
      - KONG_REAL_IP_HEADER=X-Forwarded-For   #增加这两行
    volumes:
      - /data/logs/kong:/var/log/kong
      - /etc/localtime:/etc/localtime
    ports:
      - "8000:8000"
      - "8443:8443"
      - "8001:8001"
      - "8444:8444"
    expose:
      - "8000"
      - "8443"
      - "8001"
      - "8444"
    networks:
      - dev-net
    restart: always
    depends_on:
        - kong-database
        - kong-migration
        - kong-migration-finish
```



### 安装配置

Rate Limiting插件由Kong默认提供,所以无需自行安装.由于是针对`/merchantapi` 这个借口进行限流,所以只需配置该route,并且将插件应用到这个route下即可.由于我日常使用的python进行Kong的配置,所以这里只列出我的python配置文件中相关配置.不演示具体配置了.

> 使用kong的dashboard也可以很方便的实现配置

* 配置service

```
hsq_openapi_dev = { "name": "hsq_openapi_dev",
            "host": "kong.devapi.hsq.net",
            "port": 80,
            "protocol": "http",
            "path": "/"
          }
```

* 配置route

```
#默认转发路由
hsq_openapi_dev =   {  "service_name":"hsq_openapi_dev",
                "data":{
                "name": "hsq-openapi-dev",
                "hosts": "m.devapi.hsq.net",
                "strip_path": "false",
                "protocols": ["http", "https"],
                "paths": "/",
                "methods": ["GET", "POST","PUT","OPTIONS","DELETE"]}
             }


#限流接口转发路由
hsq_merchant_api_limit = {  "service_name":"hsq_openapi_dev",
                    "data":{
                    "name": "hsq_merchant_api_limit",
                    "hosts": "m.devapi.hsq.net",
                    "strip_path": "false",
                    "protocols": ["http", "https"],
                    "paths": "/merchantapi",
                    "methods": ["GET", "POST","PUT","OPTIONS","DELETE"]}
                 }

```

* 配置rate-limiting插件

```
hsq_merchantapi_limit = { "route_name": "hsq_merchant_api_limit",  #关联到上面的route.表示该插件作用在route级别
         "data": {
         "name": "rate-limiting", #插件名称
         "config.second": 10, # 限流.每秒10个请求
         "config.policy": "local", #限流策略
         "config.limit_by": "ip"    #针对客户端IP限流
          }
        }
```

运行python脚本,配置Kong

```
 huangyong@huangyong-Macbook-Pro  ~/Desktop/kong-python   master ●✚  python3 kong.py
请输入你要配置的Kong的环境名称,例如:dev,beta,prod:>>>dev
正在创建service:hsq_openapi_dev
service:hsq_openapi_dev创建成功
开始创建routes:hsq-openapi-dev
routes路由hsq-openapi-dev创建成功
开始创建routes:hsq_merchant_api_limit
routes路由hsq_merchant_api_limit创建成功
plugins:rate-limiting创建成功.绑定在route路由:hsq_merchant_api_limit中
```



### 压测效果

为了验证插件效果,这里使用`ab` 这个简单的压测工具进行测试.

1.开启一个终端,执行下面的命令.压测命令运行了1.18秒,只有20个请求成功响应,其余80个请求失败.这恰好符合了rate-limiting插件每秒10个请求的限流策略

> 由于是在dev环境,所有只有一个Kong节点.如果外部流量负载均衡分发到Kong集群的所有节点,那么总体的限流应该是:Kong节点数量x限流数量

```
ab -n 100 -c 10 https://m.devapi.hsq.net/merchantapi

......
Document Path:          /merchantapi
Document Length:        122 bytes

Concurrency Level:      10
Time taken for tests:   1.180 seconds
Complete requests:      100         #总共100个请求
Failed requests:        80          #失败了80个
   (Connect: 0, Receive: 0, Length: 80, Exceptions: 0)
Non-2xx responses:      80
Total transferred:      41888 bytes
HTML transferred:       5720 bytes
Requests per second:    84.71 [#/sec] (mean)
Time per request:       118.044 [ms] (mean)
Time per request:       11.804 [ms] (mean, across all concurrent requests)
Transfer rate:          34.65 [Kbytes/sec] received
......
```

2. 将请求继续增大,同时使用curl和浏览器访问该域名.发现请求被拒绝

```
 huangyong@huangyong-Macbook-Pro  ~  curl https://m.devapi.hsq.net/merchantapi
{
  "message":"API rate limit exceeded"
}%
```

![](https://img2.jesse.top/20210119164349.png)



3. 在压测的同时,使用另外一个客户端来同时访问该接口,可以正常访问

```
10.0.2.20 - - [19/Jan/2021:14:05:14 +0800] "GET /merchantapi HTTP/1.0" 429 41 "-" "ApacheBench/2.3"
10.0.2.20 - - [19/Jan/2021:14:05:14 +0800] "GET /merchantapi HTTP/1.0" 429 41 "-" "ApacheBench/2.3"
10.0.2.20 - - [19/Jan/2021:14:05:14 +0800] "GET /merchantapi HTTP/1.0" 429 41 "-" "ApacheBench/2.3"
10.0.99.1 - - [19/Jan/2021:14:05:14 +0800] "GET /merchantapi HTTP/1.0" 200 122 "-" "curl/7.29.0"   #其他客户端仍然可以正常访问
10.0.2.20 - - [19/Jan/2021:14:05:14 +0800] "GET /merchantapi HTTP/1.0" 429 41 "-" "ApacheBench/2.3"
10.0.2.20 - - [19/Jan/2021:14:05:14 +0800] "GET /merchantapi HTTP/1.0" 429 41 "-" "ApacheBench/2.3"
10.0.2.20 - - [19/Jan/2021:14:05:14 +0800] "GET /merchantapi HTTP/1.0" 429 41 "-" "ApacheBench/2.3"
```

