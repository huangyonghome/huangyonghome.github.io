---
title: nginx安装Lua模块
date: 2018-08-3 11:59:58
tags:  [nginx, web ]
categories: Linux-Web
comments: true
copyright: true
---

## 安装Lua模块

#### 介绍

nginx编译安装lua模块方法,在网上能搜到许多的文档.但是都千篇一律.根据笔者的实际配置经历,网上的文档在我们的服务器上编译会报错,原因是nginx的lua模块版本(网上文档都是采用v0.10.11版本的lua-nginx-module模块)和nginx不兼容,我尝试过Nginx1.12,1.13,1.17等多个版本均无法安装.

所以笔者最终采用的的lua-nginx-module模块是0.10.9rc7版本,这才成功编译安装lua模块.

>  编译过程中虽然会覆盖现有Nginx,但是基本上属于平滑过渡,不会影响Nginx正常运行.

---

#### 服务器现有环境介绍:

1.nginx版本:1.12

2.nginx安装方式: yum安装

<!--more-->

---

#### 安装步骤:

##### 1.安装luaJIT模块

```
cd /usr/local/src
wget http://luajit.org/download/LuaJIT-2.1.0-beta2.tar.gz
tar zxf LuaJIT-2.1.0-beta2.tar.gz
cd LuaJIT-2.1.0-beta2

make PREFIX=/usr/local/luajit
make install PREFIX=/usr/local/luajit
```

##### 2.安装ngx_devel_kit模块

```
cd /usr/local/src
wget https://github.com/simpl/ngx_devel_kit/archive/v0.2.19.tar.gz
tar -xzvf v0.2.19.tar.gz
```

##### 3. 安装nginx_lua模块,注意,最好是0.10.9rc7版本

```
cd /usr/local/src
wget https://github.com/openresty/lua-nginx-module/archive/v0.10.9rc7tar.gz
tar -zxvf v0.10.9rc7.tar.gz
```

##### 4. 执行下列命令

```
sudo echo "export LUAJIT_LIB=/usr/local/luajit/lib" >> /etc/profile
sudo echo "export LUAJIT_INC=/usr/local/luajit/include/luajit-2.1" >> /etc/profile
```

##### 5.查看当前nginx模块环境

```
nginx -V

configure arguments: --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie'
```

##### 6.下载1.17版本的nginx源码(也可以使用其他版本的Nginx)

```
cd ~/
wget 'http://nginx.org/download/nginx-1.17.0.tar.gz'
tar -zxvf nginx-1.17.0.tar.gz
cd nginx-1.17.0
```

##### 7.重新编译nginx.在第五步的nginx编译参数最后加上下面lua编译参数

```
--add-module=/usr/local/src/ngx_devel_kit-0.2.19 --add-module=/usr/local/src/lua-nginx-module-0.10.9rc7

#编译完后开始安装
make -j2
make install
```

##### 8 检查nginx 版本和模块

```
#检查Nginx新版本
[work@rainbow ~]$ nginx -v
nginx version: nginx/1.17.0

#检查lua模块是否生效.最后2行表示lua模块已经编译到Nginx中
[work@rainbow ~]$ nginx -V
nginx version: nginx/1.17.0
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-36) (GCC)
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie' --add-module=/usr/local/src/ngx_devel_kit-0.2.19 --add-module=/usr/local/lua-nginx-module-0.10.9rc7
```

##### 9.如果上一步执行有问题则执行下列命令

```
echo "/usr/local/LuaJIT/lib" >> /etc/ld.so.conf
ldconfig
```

##### 10.测试lua是否支持

```
在Nginx配置文件中加入下列配置

location /hello_lua { 
      default_type 'text/plain'; 
      content_by_lua 'ngx.say("hello, lua")'; 
}

#访问nginx测试是否返回hello lua
```



