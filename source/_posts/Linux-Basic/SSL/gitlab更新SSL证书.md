---
title: gitlab自带nginx更新SSL证书
date: 2018-10-15 11:59:58
tags:  SSL
categories: [Linux-Basic,SSL ]
comments: true
copyright: true
---

## gitlab自带nginx更新SSL证书

### 背景

由于letsencrypt证书有效期太短.gitlab不是很方便自动续约.所以在阿里云上购买证书,然后下载到到服务器.(由于gitlab底层是基于nginx,所以下载证书也是选择Nginx格式)

---

找出gitlab证书存放位置:

1.在gitlab配置文件下

```
root@gitlab:~# cd /etc/gitlab
root@gitlab:/etc/gitlab# ls
gitlab.rb  gitlab.rb.bak  gitlab.rb.bak2  gitlab-secrets.json  trusted-certs

```

2.打开gitlab.rb配置文件

```
root@gitlab:/etc/gitlab# vim gitlab.rb

#可以找到下列两行配置.这个就是gitlab的ssl证书存放路径
nginx['ssl_certificate'] = "/etc/ssl/private/gitlab.pem"
nginx['ssl_certificate_key'] = "/etc/ssl/private/gitlab.key"

```

在gitlab的nginx配置文件下也可以找到相关配置

```
root@gitlab:/etc/gitlab# cd /var/opt/gitlab/nginx/conf/
root@gitlab:/var/opt/gitlab/nginx/conf# ls
gitlab-http.conf  gitlab-http.conf.bak  nginx.conf  nginx-status.conf

#在gitlab-http.conf的nginx配置文件中也定义了ssl路径:

root@gitlab:/var/opt/gitlab/nginx/conf# cat gitlab-http.conf | grep ssl_certificate
  ssl_certificate /etc/ssl/private/gitlab.pem;
  ssl_certificate_key /etc/ssl/private/gitlab.key;
  
```

3.将阿里云下载下来的证书上传到gitlab的/etc/ssl/private目录中

<!--more-->

4.重命名目前的证书文件为*.bak 将新的证书文件(.pem和.key)命名为gitlab.key和gitlab.pem

```
root@gitlab:/etc/ssl/private# ll
total 28
drwx--x--- 2 root ssl-cert 4096 Dec 18 18:30 ./
drwxr-xr-x 4 root root     4096 Jan 17  2018 ../
-rwxr-xr-x 1 root root     1679 Dec 18 18:29 gitlab.key*
-rwxr-xr-x 1 work work     1675 Dec 25  2017 gitlab.key.bak*
-rwxr-xr-x 1 root root     3663 Dec 18 18:29 gitlab.pem*
-rwxr-xr-x 1 work work     3313 Dec 25  2017 gitlab.pem.bak*
-rw-r----- 1 root ssl-cert 1704 Jun  9  2017 ssl-cert-snakeoil.key

```

5.重启gitlab程序

```
root@gitlab:/etc/ssl/private# gitlab-ctl restart
ok: run: gitaly: (pid 1442) 1s
ok: run: gitlab-monitor: (pid 1449) 1s
ok: run: gitlab-workhorse: (pid 1452) 0s
ok: run: logrotate: (pid 1463) 1s
ok: run: nginx: (pid 1469) 0s
ok: run: node-exporter: (pid 1478) 0s
ok: run: postgres-exporter: (pid 1484) 1s
ok: run: postgresql: (pid 1494) 0s
ok: run: prometheus: (pid 1503) 1s
ok: run: redis: (pid 1517) 1s
ok: run: redis-exporter: (pid 1521) 0s
ok: run: sidekiq: (pid 1532) 0s
ok: run: unicorn: (pid 1537) 1s
root@gitlab:/etc/ssl/private#
```

重新登录gitlab发现新的证书已经生效



  