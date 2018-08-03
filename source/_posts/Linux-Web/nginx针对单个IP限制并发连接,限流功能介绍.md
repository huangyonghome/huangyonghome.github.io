---
title: nginx针对单个IP限制并发连接,限流
date: 2018-08-3 11:59:58
tags:  [nginx, web ]
categories: [Linux-Web]
comments: true
copyright: true
---


## nginx针对单个IP限制并发连接,限流实战

### 简介

最近在流量高峰期时,后端数据库服务器面临非常大的压力.mysql被大量的update.select语句挤爆,慢查询记录能瞬间达千条以上.导致业务不能正常访问.监控平台不断报警.

nginx有自带的模块可以针对每一个IP限制并发连接数..还可以限制每一个IP在单位时间内的request请求频率.

<!--more-->

---

### 模块介绍

- **ngx_http_limit_conn_module** :  [官方文档](http://nginx.org/en/docs/http/ngx_http_limit_conn_module.html#limit_conn_zone)  Used to limit the number of connections per the defined key, in particular, the number of connections from a single IP address. 

  **limit_conn模块用来限制IP的并发请求连接.**

- **ngx_http_limit_req_module** :  [官方文档](http://nginx.org/en/docs/http/ngx_http_limit_req_module.html)  Used to limit the request processing rate per a defined key, in particular, the processing rate of requests coming from a single IP address 

  **limit_req模块用来限制IP在单位时间内request请求频率**

可以看到,limit_conn模块用来限制单个IP的并发连接.limit_req模块用来限制单IP入站请求的频率

---

### 实战环境

nginx版本: nginx/1.12.2

操作系统: Cetntos 7.4

拓扑: 阿里云SLB+3台nginx ECS服务器

---

### 配置

### 一.limit_conn模块限制IP并发连接

配置方法非常简单.

1.在nginx.conf的http字段中添加以下一行配置:

```
http {
   limit_conn_zone $http_x_forwarded_for zone=perip:10m;
   .....
}
```

命令格式:

limit_conn_zone :指令.在Nginx 1.1.8以后用此指令替代了此前的limit_conn指令

zone: 表示一个命名空间.

perip:表示一个名字,这个名字可以随意取.

10m: 存储IP地址的内存地址空间,一个Ipv4的地址大概占据64bit的空间.详情可见官方文档说明

$http_x_forwarded_for: 这个是根据实际的日志格式来获取IP地址.一般情况下这里应该是$remote_addr.



但是我们的日志格式为:

```
http {

	  log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" "$http_cookie" 		"$http_user_agent"'
                      '$remote_addr $server_addr $upstream_addr $host'
                      '"$http_x_forwarded_for" $upstream_response_time "$request_time"';
....
}
```



日志记录格式:

```
100.117.85.149 - - [03/Aug/2018:13:15:21 +0800] "GET / HTTP/1.0" 200 28632 "-" "-" "ApacheBench/2.3"100.117.85.149 10.27.3.27 unix:/run/php-fpm/php-fpm.sock "27.115.51.xx" 0.237 "0.237"
```

在我们的日志格式里.第一个IP字段(100.117.85.149)是个私网地址,是阿里的SLB的反代地址.而倒数第三个字段的IP地址(我隐去了最后一位IP字段)才是真正的阿里云SLB传递的客户端真实IP.

所以需要根据http_x_forwarded_for字段来限制IP



如果nginx是作为最前端的web服务器.那么默认的main日志格式应该是:

```
http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
}
```

日志记录:

```
10.0.4.241 - - [03/Aug/2018:02:20:41 +0800] "GET / HTTP/1.0" 200 225 "-" "ApacheBench/2.3" "-"
10.0.4.241 - - [03/Aug/2018:02:20:41 +0800] "GET / HTTP/1.0" 200 225 "-" "ApacheBench/2.3" "-"
10.0.4.241 - - [03/Aug/2018:02:20:41 +0800] "GET / HTTP/1.0" 200 225 "-" "ApacheBench/2.3" "-"
10.0.4.241 - - [03/Aug/2018:02:20:41 +0800] "GET / HTTP/1.0" 200 225 "-" "ApacheBench/2.3" "-"
10.0.4.241 - - [03/Aug/2018:02:20:41 +0800] "GET / HTTP/1.0" 200 225 "-" "ApacheBench/2.3" "-"
```

这里第一个字段remote_addr是客户端的真实IP地址.那么上面的指令应该改成:

```
http {
   limit_conn_zone $remote_addr zone=perip:10m;
   .....
}
```

> NOTE:官方文档建议使用$binary_remote_addr替代$remote_addr.这样好处是可以节省内存存储空间.
>
> 但是我尝试过$binary_http_x_forwarded_for并不能被nginx识别:
>
> ```
> [work@tongji-1 nginx]$ sudo nginx -t
> nginx: [emerg] unknown "binary_http_x_forwarded_for" variable
> nginx: configuration file /etc/nginx/nginx.conf test failed
> ```



由此可见.该指令需要根据实际日志格式中,真实IP所在的字段位置和变量名来具体配置.



2.在虚拟主机配置文件中调用该指令:

```
#隐去了敏感配置信息
[work@tongji-1 nginx]$ vim conf.d/tongji.conf

server {
   ......
   location / {
        limit_conn perip 5;
        root   /data/apps/piwik;
        index  index.php;
    }
    location ~ \.php$ {
        limit_conn perip 5;
        root           /data/apps/piwik;
        fastcgi_pass   unix:/run/php-fpm/php-fpm.sock;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }

}
```

指令格式:

**limit_conn perip 5;** 

**limit_conn:** 模块指令,改指令可以用在**http,server以及location字段**.代表的作用域分别是:全局nginx,全局server_name,以及某个路径

**perip:**调用nginx.conf的perip zone

5: 允许最大的并发数为5

---

**测试效果**

在另外一台服务器用ab进行一次测试:

```
[root@kong ~]# ab -n 50 -c 20 http://www.xxxxxx.com/
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking tongji.doweidu.com (be patient).....done


Server Software:
Server Hostname:        www.xxxxxx.com
Server Port:            80

Document Path:          /
Document Length:        28618 bytes

Concurrency Level:      20
Time taken for tests:   2.002 seconds
Complete requests:      50
Failed requests:        29
   (Connect: 0, Receive: 0, Length: 29, Exceptions: 0)
   ......
```

ab模拟了20个并发,一共50个request请求的压测.从输出结果的failed requests字段来看,有29个请求失败.结果并不是十分精准.而且每次执行统一的步骤得出的结果并不一致.



查看nginx服务器的access访问日志输出:

```
#为了隐私,隐去了IP地址最后一位,以及网站域名

[work@tongji-1 conf.d]$ grep '27.115.51.xx' /data/logs/nginx/tongji.access.log | grep '\b503\b'

100.117.85.54 - - [03/Aug/2018:13:17:20 +0800] "GET / HTTP/1.0" 503 537 "-" "-" "ApacheBench/2.3"100.117.85.54 10.27.3.27 - www.xxxx.com"27.115.51.xx" - "0.000"
100.117.85.96 - - [03/Aug/2018:13:17:20 +0800] "GET / HTTP/1.0" 503 537 "-" "-" "ApacheBench/2.3"100.117.85.96 10.27.3.27 - www.xxxx.com"27.115.51.xx" - "0.000"
100.117.85.81 - - [03/Aug/2018:13:17:20 +0800] "GET / HTTP/1.0" 503 537 "-" "-" "ApacheBench/2.3"100.117.85.81 10.27.3.27 - www.xxxx.com"27.115.51.xx" - "0.000"
100.117.85.167 - - [03/Aug/2018:13:17:20 +0800] "GET / HTTP/1.0" 503 537 "-" "-" "ApacheBench/2.3"100.117.85.167 10.27.3.27 - www.xxxx.com"27.115.51.xx" - "0.000"
100.117.85.114 - - [03/Aug/2018:13:17:20 +0800] "GET / HTTP/1.0" 503 537 "-" "-" 
......

```

可见Nginx阻止了很多请求,并且返回503错误.这表示验证是成功.Ningx确实能阻止部分的并发请求.

> Note:
>
> 1. 根据官网文档的解释来看,一旦nginx分配给Limit_conn模块的10M内存空间消耗完,就对于所有流量的请求都返回503错误
> 2. 个人认为这个模块并不十分成熟,至少我在公司,家里的2个测试环境都没有成功.而且尝试过nginx多个版本,多个系统平台上都没有成功.

---



### 二.IP限流配置

配置方式和上文中大同小异,只是具体的指令不同.所以部分配置不再详细解释

1.在nginx.conf配置文件的http字段中添加如下一行:

```
http {

    limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
....
}
```

解析:

rate=1r/s 表示每秒的请求次数为1次.



2.在虚拟主机配置下添加一行:

```
server {

       listen 80;
       server_name www.test.com;
       root /var/www/test;
       index index.html;

       access_log /var/log/nginx/test.access.log;
       error_log  /var/log/nginx/test.error.log;

       location / {
          limit_req zone=one burst=1 nodelay;
      }
}
```

解析:

```
limit_req指令可以放在server,http或者location字段

burst: 这个字段表示允许5次的上浮.如果第1秒、2,3,4秒请求都为1个，那么第5秒的请求为2个是被允许的。
       但是如果你第1秒就2个请求，第2秒超过1的请求返回503错误。

nodelay: 如果不设置该选项，严格使用平均速率限制请求数,第1秒5个请求时，4个请求放到第2秒执行，
         设置nodelay，5个请求将在第1秒执行。
```

---



**测试**

在另外一台服务器执行访问nginx服务

```
[root@kong ~]# curl -i http://www.test.com/index.html
HTTP/1.1 200 OK
Server: nginx/1.12.2
Date: Thu, 02 Aug 2018 22:11:55 GMT
Content-Type: text/html
Content-Length: 225
Last-Modified: Wed, 27 Jun 2018 21:46:49 GMT
Connection: keep-alive
ETag: "5b3405c9-e1"
Accept-Ranges: bytes

<html>
  <head>
   <title>Welcome to ansible </title>
  </head>

  <body>
    <h1>nginx,configured by ansible</h1>
    <p>If you see this,Ansible successfully installed nginx.</p>
    <p>Ansible managed</p>
  </body>
</html>
[root@kong ~]# curl -i http://www.test.com/index.html
HTTP/1.1 503 Service Temporarily Unavailable
Server: nginx/1.12.2
Date: Thu, 02 Aug 2018 22:11:55 GMT
Content-Type: text/html
Content-Length: 213
Connection: keep-alive

<html>
<head><title>503 Service Temporarily Unavailable</title></head>
<body bgcolor="white">
<center><h1>503 Service Temporarily Unavailable</h1></center>
<hr><center>nginx/1.12.2</center>
</body>
</html>

```



测试可见,第一次请求是正常的..但是当1秒内第二次请求时就返回503错误了.提示503 Service Temporarily Unavailable



用ab测试结果如下:

```
[root@kong ~]# ab -n 10 -c 2 http://www.test.com/
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking www.test.com (be patient).....done


Server Software:        nginx/1.12.2
Server Hostname:        www.test.com
Server Port:            80

Document Path:          /
Document Length:        225 bytes

Concurrency Level:      2
Time taken for tests:   0.013 seconds
Complete requests:      10
Failed requests:        8
......
```

我个人认为这个模块还是非常准确的.发起10次请求里,有8次失败.

---



