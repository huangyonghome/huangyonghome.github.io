---
title: kong API网关介绍及API接口配置
date: 2018-11-20 12:59:58
tags: kong
categories: [Linux-分布式&消息队列 ]
comments: true
copyright: true
---

## kong API网关介绍及API接口配置

上一篇讲解了kong+cassandra的部署安装方法.接下来讲解一下kong的API配置.官网上也有详细的api介绍.

下面简单讲解一下kong的各个api组件:

<!--more-->

#### * service

Service 顾名思义，就是我们自己定义的上游服务，通过Kong匹配到相应的请求要转发的地方

Service 可以与下面的Route进行关联，一个Service可以有很多Route，匹配到的Route就会转发到Service中.
​    
当然中间也会通过Plugin的处理，增加或者减少一些相应的Header或者其他信息

Service可以是一个实际的地址，也可以是定义的一个upstream

---

#### * route

   Route 字面意思就是路由，实际就是我们通过定义一些规则来匹配客户端的请求，每个路由都会关联一个Service,并且Service可以关联多个Route，当匹配到客户端的请求时，每个请求都会被代理到其配置的Service中

Route作为客户端的入口，通过将Route和Service的松耦合，可以通过hosts path等规则的配置，最终让请求到不同的Service中

例如，我们规定api.example.com 和 api.service.com的登录请求都能够代理到123.11.11.11:8000端口上，那我们可以通过hosts和path来路由

```
1. 创建一个Service s1，其相应的host和port以及协议为http://123.11.11.11:8000

2. 创建一个Route，关联的Service为s1，其hosts为[api.service.com, api.example.com],path为login

3. 将域名api.example.com和api.service.com的请求转到到我们的Kong服务器的前面的负载均衡调度器上

那么，当我们请求api.example.com/login和api.service.com/login时，其通过Route匹配，然后转发到Service，最终将会请求我们自己的服务。
```
---

#### * upstream

和nginx的upstream概念一模一样..这是指您自己的后端真实服务器位于Kong后面，客户端请求被转发到该服务器。
相当于Kong提供了一个负载的功能，基于Nginx的虚拟主机的方式做的负载功能

当我们部署集群时，一个单独的地址不足以满足我们的时候，我们可以使用Kong的upstream来进行设置

upstream的功能和nginx也类似,可以指定后端真实的target服务器.还提供健康检查机制

---

#### * target

target 就是在upstream进行负载均衡的终端，当我们部署集群时，需要将每个节点作为一个target，并设置负载的权重，当然也可以通过upstream的设置对target进行健康检查。
​    
当我们使用upstream时，整个路线是 Route >> Service >> Upstream >> Target 

---

#### * api

用于表示您的上游服务的传统实体。自0.13.0起已经被弃用。

---

#### * consumer

Consumer 可以代表一个服务，可以代表一个用户，也可以代表消费者，可以根据我们自己的需求来定义

可以将一个Consumer对应到实际应用中的一个用户，也可以只是作为一个Service的请求消费者

---

#### * plugin

kong官方提供了各种各样的插件,比如限流,认证,签名,http远程日志等..这些插件在请求被代理到上游API之前或之后执行。例如，请求之前的Authentication或者是请求限流插件的使用

Plugin可以和Service绑定，也可以和Route以及Consumer进行关联。当和不同的kong组件进行绑定时,插件的作用范围也不同.

---

### 配置

官方提供了详细的配置参数 [admin api](https://docs.konghq.com/0.14.x/admin-api/#add-service)

可以通过curl命令行来配置各个API组件,也可以通过kong-dashboard的UI界面配置.这里演示命令行的配置方法

#### * service配置

下面的例子中注册一个service.名字是betaapi,协议是http.主机是上游的upstream后端服务器组:beta_background_server

```
work@DWD-BETA conf]$ curl  -X POST --url http://localhost:8001/services/ \
> --data 'name=betaapi' \
> --data 'protocol=http' \
> --data 'host=beta_background_server'
```

执行结果

包含的信息有创建时间,service这个api组件的ID.以及一些默认参数,比如retries,write_timeout

```
{
    "host":"beta_background_server",
    "created_at":1541230207,
    "connect_timeout":60000,
    "id":"bb45687e-7cf5-48b2-b3ae-6847e28736bf",
    "protocol":"http",
    "name":"betaapi",
    "read_timeout":60000,
    "port":80,
    "path":null,
    "updated_at":1541230207,
    "retries":5,
    "write_timeout":60000
}
```
查看service配置

```
[work@DWD-BETA conf]$ curl -i -X GET http://localhost:8001/services
HTTP/1.1 200 OK
Date: Sat, 03 Nov 2018 07:54:25 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.14.1
Content-Length: 285



{
    "next":null,
    "data":[
        {
            "host":"beta_background_server",
            "created_at":1541230207,
            "connect_timeout":60000,
            "id":"bb45687e-7cf5-48b2-b3ae-6847e28736bf",
            "protocol":"http",
            "name":"betaapi",
            "read_timeout":60000,
            "port":80,
            "path":null,
            "updated_at":1541230207,
            "retries":5,
            "write_timeout":60000
        }
    ]
}

```

修改配置可以使用PATCH方法,例如将service的名字修改为BETA

修改的时候需要指定要修改的service或者route等组件的ID.

```
curl -X PATCH http://localhost:8001/services/bb45687e-7cf5-48b2-b3ae-6847e28736bf/ \
> --data 'name=beta'


{"host":"beta_background_server","created_at":1541230207,"connect_timeout":60000,"id":"bb45687e-7cf5-48b2-b3ae-6847e28736bf","protocol":"http","name":"beta","read_timeout":60000,"port":80,"path":null,"updated_at":1541231842,"retries":5,"write_timeout":60000}
```

#### * route配置

route定义为访问某个服务的路径,类似于网络设备中的路由的概念,它定义了访问某个服务或者域名(目的地)时,转交给某个service组件(类似于下一跳).

例如:访问m.betaapi.haoshiqi.net的请求就转交给刚才定义的service

```
[work@DWD-BETA conf]$ curl -X POST --url http://localhost:8001/routes/ \
> --data 'service.id=bb45687e-7cf5-48b2-b3ae-6847e28736bf' \
> --data 'protocols[]=http' \
> --data 'protocols[]=https' \
> --data 'methods[]=GET&methods[]=POST' \
> --data 'hosts[]=m.betaapi.haoshiqi.net'

```
这里我只定义了methods和protocols.没有定义path.这3个参数必须指定一个.它表示了访问目的服务(m.betaapi.haoshiqi.net)的方法和协议,或者具体路径.

我使用了2种写法:

```
--data 'protocols[]=http' \
--data 'protocols[]=https' \
```
和

```
methods[]=GET&methods[]=POST'
```

执行结果:

```
{
    "created_at":1541232959,
    "strip_path":true,
    "hosts":[
        "m.betaapi.haoshiqi.net"
    ],
    "preserve_host":false,
    "regex_priority":0,
    "updated_at":1541232959,
    "paths":null,
    "service":{
        "id":"bb45687e-7cf5-48b2-b3ae-6847e28736bf"
    },
    "methods":[
        "GET",
        "POST"
    ],
    "protocols":[
        "http",
        "https"
    ],
    "id":"d4f02c14-0482-472c-9a4f-04361dd5f579"
}
```

下面是定义了paths:

```
[work@DWD-BETA plugins]$ curl -i -XPOST --url http://localhost:8001/routes/ --data 'protocols[]=http' --data 'protocols[]=https' --data 'methods[]=GET&methods[]=POST' --data 'paths[]=/merchantapi' --data 'hosts[]=m.betapai.haoshiqi.net' --data 'service.id=bb45687e-7cf5-48b2-b3ae-6847e28736bf'
HTTP/1.1 201 Created
Date: Wed, 07 Nov 2018 10:26:00 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.14.1
Content-Length: 324

{"created_at":1541586360,"strip_path":true,"hosts":["m.betapai.haoshiqi.net"],"preserve_host":false,"regex_priority":0,"updated_at":1541586360,"paths":["\/merchantapi"],"service":{"id":"bb45687e-7cf5-48b2-b3ae-6847e28736bf"},"methods":["GET","POST"],"protocols":["http","https"],"id":"a2bb14fd-6423-45bb-be60-5481bc5a3038"}
```
> route的hosts字段不仅可以指定具体的域名,还可以配置通配符域名,或者更高级的正则写法.

---

#### * upstream配置

upstream的名字应该要和service的host匹配

可以看到定义upstream时,kong提供了很多默认参数

```
[work@DWD-BETA conf]$ curl -X POST --url http://localhost:8001/upstreams/ --data 'name=beta_background_server'

```
输出结果

```
{
    "healthchecks":{
        "active":{
            "unhealthy":{
                "http_statuses":[
                    429,
                    404,
                    500,
                    501,
                    502,
                    503,
                    504,
                    505
                ],
                "tcp_failures":0,
                "timeouts":0,
                "http_failures":0,
                "interval":0
            },
            "http_path":"/",
            "healthy":{
                "http_statuses":[
                    200,
                    302
                ],
                "interval":0,
                "successes":0
            },
            "timeout":1,
            "concurrency":10
        },
        "passive":{
            "unhealthy":{
                "http_failures":0,
                "http_statuses":[
                    429,
                    500,
                    503
                ],
                "tcp_failures":0,
                "timeouts":0
            },
            "healthy":{
                "successes":0,
                "http_statuses":[
                    200,
                    201,
                    202,
                    203,
                    204,
                    205,
                    206,
                    207,
                    208,
                    226,
                    300,
                    301,
                    302,
                    303,
                    304,
                    305,
                    306,
                    307,
                    308
                ]
            }
        }
    },
    "created_at":1541233739783,
    "hash_on":"none",
    "id":"111dbaf7-0283-49ae-958e-7146ee0e4e33",
    "hash_on_cookie_path":"/",
    "name":"beta_background_server",
    "hash_fallback":"none",
    "slots":10000
}
```
---

#### *target配置

定义target时需要指定upstreams组件路径下,而且需要指定在哪个upstream的ID下配置targets.


>注意:后端的target真实服务器无论是域名还是IP地址,都需要指定端口号,默认端口号是8000

```
[work@DWD-BETA conf]$ curl -X POST --url http://localhost:8001/upstreams/111dbaf7-0283-49ae-958e-7146ee0e4e33/targets --data 'target=kong.beta.haoshiqi.net:80'
```

```
{
    "created_at":1541234259931,
    "id":"a40fb1b5-a739-4de7-8208-9eaa52c03c5e",
    "upstream_id":"111dbaf7-0283-49ae-958e-7146ee0e4e33",
    "target":"kong.beta.haoshiqi.net:80",
    "weight":100
}
```

这里我定义的target真实服务器是kong.beata.haoshiqi.net.这样一来访问m.betaapi.haoshiqi.net域名经过routes组件转交给services组件.然后services反代到upstreams里的真实target服务器.最终将请求转交到kong.beta.haoshiqi.net服务器

---


