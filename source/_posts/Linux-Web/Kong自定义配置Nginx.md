---
title: Kong自定义配置Nginx
date: 2021-01-19 11:59:58
tags:  kong
categories: [Linux-Web, kong]
comments: true
copyright: true
---



## Kong自定义配置Nginx



### 介绍

Kong是基于Nginx实现代理转发.官方的 `nginx.conf` 配置文件过于简单.如果需要优化nginx的性能,就需要修改默认的nginx配置文件,或者重新自定义一个nginx配置文件.

具体方法可以参考官方文档: https://docs.konghq.com/2.2.x/configuration/#environment-variables



下面介绍2种方式自定义nginx的配置



### 通过环境变量注入

Kong服务启动时会每次都新建一个新的nginx配置文件.可以通过将nginx指令注入到 `kong.conf` 配置文件中从而配置到这个新的nginx配置文件

<!--more--> 

#### 注入Nginx单个指令



注入到Kong的环境变量一般包含下面2种前缀.前缀名不同代表注入的nginx指令作用在不同的作用域下.Kong会将环境变量的前缀去掉,然后将环境变量的后面部分注入到nginx.

- `nginx_http_` 该前缀环境变量会被注入到Nginx的http代码块
- `nginx_proxy_` 该前缀会被注入到nginx的server代码块



例如.如果注入以下环境变量到 `kong.conf` 配置文件:

```
nginx_proxy_large_client_header_buffers=16 128k
```

Kong会将以下环境变量注入到Nginx配置文件的代理 `server` 块中

```
large_client_header_buffers 16 128k;
```

下面的环境变量,会被注入到nginx的http块中

```
export KONG_NGINX_HTTP_OUTPUT_BUFFERS="4 64k"

#注入以下Nginx指令
output_buffers 4 64k;
```

> 还有一种前缀 `Nginx_admin_` 这个作用在kong的admin api,所以用的较少



#### 注入Nginx代码块



对于一些复杂的配置场景,比如需要将整个server代码块添加到Nginx配置文件.可以使用上面的环境变量注入的方式,注入一个 `include` 指令到Nginx配置文件.

例如下面这个nginx的server代码块文件.假如该文件名为 `my-server.conf` 

```
# custom server
server {
  listen 2112;
  location / {
    # ...more settings...
    return 200;
  }
}
```

可以通过下面的方式添加到 `kong.conf` 配置文件

```
nginx_http_include = /path/to/your/my-server.conf
```

或者通过环境变量方式注入

```
export KONG_NGINX_HTTP_INCLUDE="/path/to/your/my-server.conf"
```

这样当Kong启动后,server代码块会被添加到Nginx的配置文件.

> 这里也可以使用相对路径来注入一个server代码块的配置文件,但是配置文件需要在 `kong.conf` 配置文件的prefix路径之下.或者kong启动时候通过 `-p` 参数自定义的prefix路径之下



### 自定义Nginx模板

kong在启动的时候会根据 `/usr/local/share/lua/5.1/kong/templates/nginx.lua`和 `/usr/local/share/lua/5.1/kong/templates/nginx_kong.lua` 这2个lua模板来自动生成nginx的配置文件.当Kong启动后会自动在prefix路径下生成 `nginx.conf` 和 `nginx-kong.conf` .前者是Nginx的主配置文件,然后通过include方式引入了 `nginx_kong.conf` 



当kong启动后,会产生下面2个文件

```
/usr/local/kong
    - nginx.conf
    - nginx-kong.conf
```

> 在 https://github.com/kong/kong/tree/master/kong/templates下也存放了kong的默认模板文件.



所以在 `usr/local/kong` 目录下直接修改 `Nginx.conf` 配置文件无法永久生效.当kong重启时,配置文件会被默认的Lua目标所覆盖和替代

如果一定要自定义nginx配置文件.可以自定义nginx的模板文件来替代 `Nginx.lua` .然后在该模板文件里引入 `nginx-kong.conf` 



#### 实现步骤

1. 拷贝 `nginx.conf` 配置文件为  `nginx.conf.template` 

```
cp /usr/local/kong/nginx.conf nginx.conf.template
```



1. 自定义配置 `nginx.conf.template` .例如下面是我的配置文件内容

```
pid pids/nginx.pid;
error_log logs/error.log ${{LOG_LEVEL}}; 

# injected nginx_main_* directives
daemon off;
worker_processes auto;
worker_rlimit_nofile 204800;

events {
    # injected nginx_events_* directives
    multi_accept on;
    use epoll;
    worker_connections  204800;
}

http {
    default_type  application/octet-stream;
    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    include 'nginx-kong.conf';

    keepalive_timeout  60;
    keepalive_requests 1024;
    client_header_buffer_size 4k;
    large_client_header_buffers 4 32k;

    types_hash_max_size 2048;
    client_body_timeout 180;
    client_header_timeout 10;
    send_timeout 240;

    proxy_connect_timeout   1000ms;
    proxy_send_timeout      5000ms;
    proxy_read_timeout      5000ms;
    proxy_buffers           64 8k;
    proxy_busy_buffers_size    128k;
    proxy_temp_file_write_size 64k;
    proxy_redirect off;
    proxy_next_upstream off;
    
    gzip on;
    gzip_min_length 1k;
    gzip_buffers 4 16k;
    gzip_http_version 1.0;
    gzip_comp_level 2;
    gzip_types text/plain application/x-javascript text/css application/xml;
    gzip_vary on;
}
```

> 注意该配置文件内的指令不能和 `nginx-kong.conf` 配置文件有同名或者冲突.否则kong无法启动



1. 重新启动Kong.使用下面的参数指定自定义的Nginx模板文件

```
kong start -c /etc/kong/kong.conf --nginx-conf nginx.conf.template
```

####  

#### docker运行kong



如果是docker方式运行.可以使用 `Dockerfile` 自定义kong镜像

以下是Dockerfile文件内容

```
FROM kong:2.2.0
COPY nginx.conf.template /usr/local/kong/nginx.conf.template
CMD kong start --nginx-conf /usr/local/kong/nginx.conf.template
```

编译docker镜像

```
docker build -t dwd-kong:2.2.0 .
```



重启运行docker.但是要先在kong容器运行 `kong migrations up` 和 `kong migrations finish` 命令.所以 `docker-compose.yml` 配置文件内容如下

```
kong:
    image: hub.doweidu.com/beta/dwd-kong:2.2.0
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
      - KONG_TRUSTED_IPS=0.0.0.0/0,::/0
      - KONG_REAL_IP_HEADER=X-Forwarded-For
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
        
kong-migration:
    image: hub.doweidu.com/beta/dwd-kong:2.2.0
   # command: "kong migrations bootstrap"
    command: "kong migrations up"
    networks:
      - dev-net
    restart: on-failure
    environment:
      KONG_PG_HOST: kong-database
    depends_on:
      - kong-database

  kong-migration-finish:
    image: hub.doweidu.com/beta/dwd-kong:2.2.0
   # command: "kong migrations bootstrap"
    command: "kong migrations finish"
    networks:
      - dev-net
    restart: on-failure
    environment:
      KONG_PG_HOST: kong-database
    depends_on:
      - kong-database
      - kong-migration
```

启动容器后.可以查看配置文件是否生效:

```
[work@docker docker-compose]$docker exec kong cat /usr/local/kong/nginx.conf
pid pids/nginx.pid;
error_log logs/error.log notice;

# injected nginx_main_* directives
daemon off;
worker_processes auto;
worker_rlimit_nofile 204800;

events {
    # injected nginx_events_* directives
    multi_accept on;
    use epoll;
    worker_connections  204800;
}

http {
    default_type  application/octet-stream;
    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    include 'nginx-kong.conf';
 .......略...........
```

如此,便实现了自定义kong的nginx配置文件,这在大并发场景中可能需要优化nginx的转发性能.如果是小规模场景中,可以使用Kong的默认的Nginx配置文件即可.