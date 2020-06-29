---
title: nginx set命令——TradeCenter中心Nginx配置
date: 2018-08-3 11:59:58
tags:  [nginx, web ]
categories: [Linux-Web]
comments: true
copyright: true
---



## nginx set命令——TradeCenter中心Nginx配置

#### 交易中心接口有个需求..

/open/callback/http 这个路径允许外部调用.会有一些外部服务来调用我们的交易中心接口

除此之外互联网访问交易中心域名其他所有资源都拒绝.

目前nginx的配置文件如下:

```
#隐去多余的配置信息
work@docker ~]$cat /data/conf/nginx/conf.d/dwd-trade-ssl.conf
server {

  server_name trade.dev.xxxxx.com;

  location  / {
    index index.php index.html;
    add_header Cache-Control no-cache;

    if (!-e $request_filename) {
       rewrite ^/(.*)  /index.php?$1 last;
    }
  }


  location ~ .*\.(php|php5)?$ {

         fastcgi_pass  php:9000;
         include fastcgi_params;
         fastcgi_param  SCRIPT_FILENAME    $document_root$fastcgi_script_name;
         fastcgi_param  HTTPS              off;
         fastcgi_buffer_size 128k;
         fastcgi_buffers 256 16k;

         fastcgi_busy_buffers_size 256k;
         fastcgi_temp_file_write_size 256k;
   }

}
```

<!--more-->

### 配置思路

这个看似简单的需求,但是配置过程中踩过N多的坑,下面一一道来:



**1.添加个location.然后在根路径下使用deny语句:**

```
  location  /open/callback/http  {
    index index.php index.html;
    add_header Cache-Control no-cache;

    if (!-e $request_filename) {
       rewrite ^/(.*)  /index.php?$1 last;
    }
  }

  location  / {
    allow 10.0.0.0/8;
    allow 127.0.0.1;
    deny all;
  }
```

但是由于/open/callback/http是个虚拟路径,最后会被rewrite到根目录下的index.php.(实际上访问任何目录都是访问/index.php)

所以到最后所有请求都会到location /下,无论来自公网的请求是否访问/open/callback/http的URL都会被deny all;



**2.通过set打一个tag**

```
 #定义一个全局的变量
 set $ban_url 1;
 
 location  /open/callback/http  {
 
    #如果是访问这个路径,则设置为0
    set $ban_url 0;
    
    index index.php index.html;
    add_header Cache-Control no-cache;

    if (!-e $request_filename) {
       rewrite ^/(.*)  /index.php?$1 last;
    }
  }

  location  / {
    #如果$ban_url为True.即不为0..则拒绝外网访问
    if ($ban_url) {
    allow 10.0.0.0/8;
    allow 127.0.0.1;
    deny all;      
    }
  
  }
```

这个思路连nginx的语法测试都没有通过,因为If条件判断下不允许使用allow,deny指令



**3.既然不允许allow IP. 那就自行判断IP来源**

```
 #定义一个全局的变量
 set $ban_url 1;
 
 location  /open/callback/http  {
 
    #如果是访问这个路径,则设置为0
    set $ban_url 0;
    
    index index.php index.html;
    add_header Cache-Control no-cache;

    if (!-e $request_filename) {
       rewrite ^/(.*)  /index.php?$1 last;
    }
  }

  location  / {
    #如果$ban_url为True.即不为0.而且IP也不是10网段的内网IP,则拒绝外网访问
    if ($ban_url) and ($remote_addr !~ "^10\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$") {
       return 403; 
    }
  
  }
```

同样无法通过Nginx的语法检测,因为Nginx不支持if的 AND OR多重条件判断.

----

### 最终方案

既然不支持多重判断,那还是只能通过打tag的方式来判断

```
 #定义一个全局的变量
 set $ban_url 1;
 
 location  /open/callback/http  {
 
    #如果是访问这个路径,则设置为0
    set $ban_url 0;
    
    index index.php index.html;
    add_header Cache-Control no-cache;

    if (!-e $request_filename) {
       rewrite ^/(.*)  /index.php?$1 last;
    }
  }

  location  / {
    #如果IP不是10网段的内网IP,则打一个$ban_rul+1的标签
    if ($remote_addr !~ "^10\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$") {
         set $ban_url "${ban_url}1";
    }
  
  }
  
  #如果ban_url是1,而且又不是内网IP.则表示从互联网请求非/open/callback/http下的资源.则直接拒绝
   if ($ban_url = "11") {
      return 403;

   }
```

---

###  测试

```
#从互联网访问URL,返回403

huangyong@huangyong-Macbook-Pro  ~  curl -I https://trade.beta.xxxxx.com/open
HTTP/2 403
server: nginx
date: Tue, 28 May 2019 05:58:40 GMT
content-type: text/html
content-length: 162

 huangyong@huangyong-Macbook-Pro  ~  curl -I https://trade.beta.xxxxx.com/
HTTP/2 403
server: nginx
date: Tue, 28 May 2019 05:58:43 GMT
content-type: text/html
content-length: 162

#从互联网访问open/callback/http/下资源返回200

 huangyong@huangyong-Macbook-Pro  ~  curl -I https://trade.beta.xxxxx.com/open/callback/http/wxpay
HTTP/2 200
server: nginx
date: Tue, 28 May 2019 06:03:49 GMT
content-type: application/octet-stream
content-length: 640
last-modified: Fri, 17 May 2019 01:57:28 GMT
etag: "5cde1508-280"
cache-control: no-cache
accept-ranges: bytes
```

---

### 最终配置参考:

```
[work@DWD-BETA conf.d]$cat dwd-trade-ssl.conf
server {

  server_name trade.beta.xxxxx.com;
  root /data/apps/dwd-trade-beta/current/public;

  set $ban_url 1;

  location /open/callback/http {
    set $ban_url 0;
    index index.php index.html;
    add_header Cache-Control no-cache;
    rewrite ^/(.*) /index.php?$1 last;

}

  location  / {
    if ($remote_addr !~ "^10\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$") {
          set $ban_url "${ban_url}1";

   }
   if ($ban_url = "11") {
      return 403;

   }

    index index.php index.html;
    add_header Cache-Control no-cache;

    if (!-e $request_filename) {
       rewrite ^/(.*)  /index.php?$1 last;
    }
  }

  location ~ .*\.(php|php5)?$ {

         fastcgi_pass unix:/data/run/php-fpm71.sock;
         include fastcgi_params;
         fastcgi_param  SCRIPT_FILENAME    $document_root$fastcgi_script_name;
         fastcgi_param  HTTPS              off;
         fastcgi_buffer_size 128k;
         fastcgi_buffers 256 16k;

         fastcgi_busy_buffers_size 256k;
         fastcgi_temp_file_write_size 256k;
	 fastcgi_connect_timeout 180;
         fastcgi_read_timeout  600;
         fastcgi_send_timeout 600;
   }


}
```



----



### 另外一个问题

今天又遇到一个类似的需求.交易中心要求https://m.betaapi.xxxxx.com/open/order/notify接口只允许内网地址访问.

经过测试发现上面的那个配置在这个场景下不管用.

在BETA服务器的msf-open-api.conf配置文件的内加入下列一行.显示ban_url的值:

```
location ~ .*\.(php|php5)?$ {
         #fastcgi_pass  127.0.0.1:9000;
         fastcgi_pass unix:/data/run/php-fpm71.sock;
         include fastcgi_params;
         add_header ban-url $ban_url always; #添加这一行
         .....略......
 }
```

在我电脑终端上请求上面的URL

```
 huangyong@huangyong-Macbook-Pro  ~  curl -I  https://m.betaapi.xxxxx.com/open/order/notify
HTTP/2 200  #从外网居然可以正常访问
date: Mon, 03 Jun 2019 10:07:35 GMT
content-type: application/html;charset=utf-8
access-control-allow-headers: Content-Type
access-control-allow-origin: http://.xxxxx.com
access-control-allow-credentials: true
server: apache/1.8.0
x-powered-by: PHP
ban-url: 1   #这里值是1.而不是11 
```

经过反复测试发现if语句要定义在location php下:

```
  location ~ .*\.(php|php5)?$ {

#添加下面2个If判断语句

    if ($remote_addr !~ "^10\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$") {
       set $ban_url "${ban_url}1";

   }
   if ($ban_url = "11") {
      return 403;

   }
   ......略..........

   }
```

再次测试发现可以成功的匹配到ban_url变量的值

```
 huangyong@huangyong-Macbook-Pro  ~  curl -I  https://m.betaapi.xxxxx.com/open/order/notify
HTTP/2 403  #从互联网访问return 403
server: nginx
date: Mon, 03 Jun 2019 10:09:32 GMT
content-type: text/html
content-length: 162
ban-url: 11  #正常匹配到值
```

由此可见下面定义的这个rewrite语句没有重定向到location /根目录下.而是直接重定向到location php

```
set $ban_url 0;

 location ^~ /open/order/notify {
       set $ban_url 1;
       index index.php index.html;
       #add_header ban-url $ban_url always;
       add_header Cache-Control no-store;
      rewrite ^/(.*)  /index.php?$1 last;

}
```



还有个更简单的配置.直接在path中返回403,.不需要打tag,也不需要转发到根路径或者后端PHP:

```
location  /service  {

   #如果IP不是办公室出口IP或者内网IP,则返回403
  if ($remote_addr !~ "27.115.51.166|^10\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$") {
           return 403;
   }

    add_header Cache-Control no-cache;
    rewrite ^/(.*)  /index.php?$1 last;
  }




  location / {

    index index.php index.html;
    add_header Cache-Control no-cache;

    if (!-e $request_filename) {
       rewrite ^/(.*)  /index.php?$1 last;
    }
  }

```



---



#### 注意:

 配置完成后,在阿里云SLB里还要配置一下健康检查,将4xx的状态码配置为正常状态码,不然阿里云访问trade.xxxxx.com域名返回403会认为后端服务器异常,从而关闭流量转发.导致所有流量都被SLB屏蔽了.


