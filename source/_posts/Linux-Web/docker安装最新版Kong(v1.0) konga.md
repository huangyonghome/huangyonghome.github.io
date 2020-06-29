---
title: docker安装最新版Kong(v1.0)+konga
date: 2020-06-26 11:59:58
tags:  kong
categories: [Linux-Web, kong]
comments: true
copyright: true
---



### docker安装最新版Kong(v1.0)+konga

参考以下文档:

[Kong installation](https://docs.konghq.com/install/docker/?_ga=2.167535422.1288669860.1553147426-917309945.1539077269)

[konga github](https://github.com/pantsel/konga#installation)

---



#### docker安装kong+postgresql



1.创建一个docker网络用于docker,postgresql和konga容器间通信

```
docker network create kong-net
```



2.启动posgtresql容器

```
docker run -d --name kong-database \
               --network=kong-net \
               -p 5432:5432 \
               -e "POSTGRES_USER=kong" \
               -e "POSTGRES_DB=kong" \
               postgres:9.6
```



3.初始化postgresql数据库

```
 $ docker run --rm \
     --network=kong-net \
     -e "KONG_DATABASE=postgres" \
     -e "KONG_PG_HOST=kong-database" \
     -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
     kong:latest kong migrations bootstrap
```

> 注意两点:
>
> 1.最好是先删除本地的kong镜像.因为本地的Kong:lastest镜像不一定就是最新版
>
> 2.如果本地的kong:latest镜像地域0.15版本,则不支持bootstrap命令.可以将bootstrap命令替换成up

<!--more-->

4.启动kong容器

```
docker run -d --name kong \
     --network=kong-net \
     -v /data/logs/kong:/var/log/kong \
     -v /data/apps/kong/plugins/:/usr/local/share/lua/5.1/kong/plugins/ \
     -e "KONG_DATABASE=postgres" \
     -e "KONG_PG_HOST=kong-database" \
     -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
     -e "KONG_PROXY_ACCESS_LOG=/var/log/kong/access.log" \
     -e "KONG_ADMIN_ACCESS_LOG=/var/log/kong/admin_access.log" \
     -e "KONG_PROXY_ERROR_LOG=/var/log/kong/error.log" \
     -e "KONG_ADMIN_ERROR_LOG=/var/log/kong/admin_error.log" \
     -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
     -p 8000:8000 \
     -p 8443:8443 \
     -p 8001:8001 \
     -p 8444:8444 \
     kong:latest
```

> 这里我映射了kong的插件目录和日志目录. 

* 注意:要先吧kong容器里的/usr/local/share/lua/5.1/kong/plugins/目录下内容复制到宿主机的/data/apps/kong/plugins/目录下.否则宿主机的空目录会覆盖容器的插件目录,导致容器无法启动.

kong容器目录拷贝到宿主机方法如下:

1.先不挂载目录启动kong容器

2.执行命令拷贝kong容器的/usr/local/share/lua/5.1/kong/plugins/ 目录到宿主机/data/apps/kong/plugins/目录下:

```
[work@docker ~]$docker cp kong:/usr/local/share/lua/5.1/kong/plugins/  /data/apps/kong/plugins/
```

> 如果不需要将容器的kong插件目录映射到宿主机的话,这一步可以不需要做



容器已经成功启动:

```
[work@docker conf.d]$docker ps | grep -E "kong|postgre"
10fc881cca4b        kong:latest                                 "/docker-entrypoin..."   About an hour ago   Up About an hour    0.0.0.0:8000-8001->8000-8001/tcp, 0.0.0.0:8443-8444->8443-8444/tcp                                             kong

afd1487e29a0        postgres:9.6                                "docker-entrypoint..."   3 hours ago         Up 3 hours          0.0.0.0:5432->5432/tcp                                                                                         kong-database
```

---

#### 安装konga

konga是管理kong的一个dashboard界面.

1.先初始化数据库.这里也是用后端的postgresql数据库.官方命令如下:

```
docker run --rm pantsel/konga:latest -c prepare -a {{adapter}} -u {{connection-uri}}
```

| argument | description                            | default |
| -------- | -------------------------------------- | ------- |
| -c       | command                                | -       |
| -a       | adapter (can be `postgres` or `mysql`) | -       |
| -u       | full database connection url           |         |

执行以下命令,初始化数据库:

```
docker run --rm pantsel/konga:latest -c prepare -a postgres -u  postgres://kong@10.0.0.250:5432/konga
```

> 这里稍微有点疑问的是数据库的connection url ..完整的connection url地址是: postgres://user:password@host:port/konga
>
>  postgres://kong@10.0.0.250:5432/konga —— 这里kong代表用户名,由于没有密码所以没有指定密码.10.0.0.250是postgresql的host主机名.konga表示初始化一个数据库

执行结果如下:

```
[work@docker ~]$docker run --rm pantsel/konga:latest -c prepare -a postgres -u  postgres://kong@10.0.0.250:5432/konga
debug: Preparing database...
Using postgres DB Adapter.
Database `konga` does not exist. Creating...
Database `konga` created! Continue...
debug: Hook:api_health_checks:process() called
debug: Hook:health_checks:process() called
debug: Hook:start-scheduled-snapshots:process() called
debug: Hook:upstream_health_checks:process() called
debug: Hook:user_events_hook:process() called
debug: Seeding User...
debug: User seed planted
debug: Seeding Kongnode...
debug: Kongnode seed planted
debug: Seeding Emailtransport...
debug: Emailtransport seed planted
debug: Database migrations completed!
```



2. 启动konga

命令格式如下:

```
$ docker run -p 1337:1337 
             --network {{kong-network}} \ // optional
             -e "TOKEN_SECRET={{somerandomstring}}" \
             -e "DB_ADAPTER=the-name-of-the-adapter" \ // 'mongo','postgres','sqlserver'  or 'mysql'
             -e "DB_HOST=your-db-hostname" \
             -e "DB_PORT=your-db-port" \ // Defaults to the default db port
             -e "DB_USER=your-db-user" \ // Omit if not relevant
             -e "DB_PASSWORD=your-db-password" \ // Omit if not relevant
             -e "DB_DATABASE=your-db-name" \ // Defaults to 'konga_database'
             -e "DB_PG_SCHEMA=my-schema"\ // Optionally define a schema when integrating with prostgres
             -e "NODE_ENV=production" \ // or 'development' | defaults to 'development'
             --name konga \
             pantsel/konga
```

执行如下命令:

```
docker run -p 1337:1337 -d \
             --network=kong-net \
             -e "DB_ADAPTER=postgres" \
             -e "DB_HOST=10.0.0.250" \
             -e "DB_PORT=5432" \
             -e "DB_USER=kong" \
             -e "DB_DATABASE=konga" \
             -e "NODE_ENV=production" \
             --name konga \
             pantsel/konga
```



容器成功启动:

```
beb70407b417        pantsel/konga                               "/app/start.sh"          2 hours ago         Up 2 hours          0.0.0.0:1337->1337/tcp                                                                                         konga
```

