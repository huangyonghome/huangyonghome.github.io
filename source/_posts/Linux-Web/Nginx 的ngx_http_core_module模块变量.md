## Nginx 配置





### ngx_http_core_module模块变量

下列表格中列出了部分的变量名及其代表的意义

|                     |                                                              |
| :------------------ | ------------------------------------------------------------ |
| $arg_PARAMETER      | http请求中某个参数的值,例如/index.html?size=100,可以用$arg_size取得100这个值 |
| $args               | HTTP请求中的完整参数,例如.在请求/index.html?_w=120&_h=120中,$args表示字符串_w=120&_h=120 |
| $binary_remote_addr | 二进制客户端地址                                             |
| $document_root      | 表示当前请求所使用的root配置项的值                           |
| $uri                | 表示当前请求的URI,不带任何参数                               |
| $document_uri       | 与$uri的含义相同                                             |
| $request_uri        | 表示客户端发来的原始URI,带完整的参数.$uri和$document_uri未必是用户的原始请求,在内部重定向后可能是重定向后的URI,而$request_uri永远不会改变,始终是客户端的原始URI |
| $host               | 表示客户端请求头部中的Host字段,如果Host字段不存在,则以实际处理的server(虚拟主机)名称代替,如果host字段带有端口,如IP:PORT,那么$host是去掉端口的,它的值为IP.$host是全小写的.和$http_HEADER中的http_host不同,http_host只是取出Host头部对应的值 |
| $http_HEADER        | 表示http请求中相应头部的值,HEADER名称全小写.例如$http_host表示请求中host头部对应的值 |
| $send_http_HEADER   | 表示返回客户端的HTTP响应中相应头部的值.HEADER名称全小写,例如$send_http_content_type表示响应中Content-Type头部对应的值 |
| $is_args            | 表示请求中的URI是否带参数,.如果带参数.$is_args值为?,否则值为空字符串 |
| $limit_rate         | 表示当前连接限速,0表示不限速                                 |
| $query_string       | 表示URI中的参数,与$args相同,然而$query_string是只读的不会改变 |
| $remote_addr        | 客户端地址                                                   |
| $remote_port        | 客户端连接使用的接口                                         |
| $request_filename   | 表示用户请求中的URI经过root或者alias转换后的文件路径         |
| $request_body       | 表示Http请求中的包体,该参数只在proxy_pass或fastcgi_pass中有意义 |
| $request_method     | 表示HTTP请求的方法名,例如GET,PUT,POST等                      |
| $scheme             | 表示http scheme,如在请求https://nginx.com/中表示https        |

---



### 反向代理服务器

下面介绍负载均衡的配置项

1.**upstream块**

**语法:** upstream name {...}

**配置块:**http

upstream块定义了一个上游服务器的集群.便于反向代理中的Proxy_pass使用.例如:

```
upstream backend {
       server www.test.com;
       server www.test1.com;
       server www.test2.com;
}

server {
    location / {
        proxy_pass http://backend
    }
    
}
```

2.**server**(这个是upstream块内的server参数,并非指server{}块)

**语法:** server name [parameters];

**配置块:** upstream



server配置项指定了一台上游服务器的名字,可以是域名,IP地址端口,UNIX句柄等,在服务器后面还可以跟下列参数.

- weight=number: 设置转发权重,默认为1.
- max_fails=number: 和fail_timeout配合使用,指在fail_timeout时间段内,如果上游服务器失败次数超过max_fails,则认为这台上游服务器不可用.max_fails默认为1,如果设置为0,不检查失败次数
- fail_timeout=time: 表示该时间段内转发失败多少次后,就认为上游服务器暂时不可用.默认为10秒
- down: 表示所在的上游服务器永久下线,只在使用Ip_hash配置项时才有用
- backup:在使用ip_hash时,这个配置是无效的,它表示所在的上游服务器只是备份服务器,只有在所有的非备份上游服务器都失效后,才会启用这台backup服务器

例如:

```
upstream backend {
    server www.test.com weight=5;
    server 127.0.0.1:8080 max_fails=3 fail_timeout=30s;
    server unix:/tmp/nginx;
}
```

3. **ip_hash**

**语法:** ip_hash;

**配置块:** upstream

ip_hash表示始终将来自某一个用户的请求反代到一台固定的上游服务器上,这有助于保持用户的session一致性.ip_hash首先根据客户端的IP地址计算出一个KEY,将KEY按照Upstream集群里的上游服务器数量进行取模,然后根据取模后的结果把请求转发到相应的上游服务器中.这样就确保了同一个客户端的请求只会转发到指定的上游服务器中.

> ip_hash与weight配置不可同时使用.另外如果upstream集群中有一台上游服务器暂时不可用,应该要用down参数标识出这台参数,而不是直接将该服务器从集群中移除,确保上游服务器的数量不变,这样才能保证转发策略的一贯性.

例如:

```
upstream backend {
    ip_hash;
    server www.test.com;
    server www.test1.com;
    server www.test2.com down;
}
```

---



**有关负载均衡的日志变量**

有些负载均衡的信息可能需要记录到access_log日志中,在定义日志格式时,可以使用负载均衡提供的变量.例如:



| $upstream_addr          | 处理请求的上游服务器地址                                  |
| ----------------------- | --------------------------------------------------------- |
| $upstream_cache_status  | 表示是否命中缓存,取值范围:MISS,EXPIRED,UPDATING,STALE,HIT |
| $upstream_status        | 上游服务器返回的响应中的Http响应码                        |
| $upstream_response_time | 上游服务器的响应时间,精度到毫秒                           |
| $upstream_http_$HEADER  | http头部,如:upstream_http_host                            |

---

### 反向代理的基本配置

**下面列出一些重要的配置参数**

**1.proxy_apss**

**语法:** proxy_pass URL:

**配置块:** location, if 

此配置将当前请求反向代理到URL参数指定的服务器上.URL可以是主机名,IP地址加端口的形式,unix句柄,以及upstream名称.例如:

```
proxy_pass http://localhost:8080/uri/;
proxy_pass http://unix:/path/to/nginx.socket:/uri/;
proxy_pass http://backend;

#也可以代理成Https:
proxy_pass https://localhost;
```

默认情况下,反向代理不会转发请求中的host头部.如果需要转发,必须加上下面的参数

```
proxy_set_header Host $host;
```

**2.proxy_pass_header**

将上游服务器的响应http头部字段转发给客户端,例如:

```
proxy_pass_header X-Accel-Redirect;
```

>  该参数和proxy_hide_header字段相反.proxy_hide_header是隐藏header字段



**3.proxy_pass_request_body on | off** 

是否向上游服务器转发Http包体部分,默认是on;

**4.proxy_pass_request_headers on |off**

是否转发http头部,默认on





.