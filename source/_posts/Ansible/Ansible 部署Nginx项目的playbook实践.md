## Ansible 部署Nginx项目的playbook实践

工作中经常需要搭建项目的业务环境.需要部署同一套nginx站点业务在联调,测试,预发布,生产等各种不同环境的服务器上.这些工作主要包括两个部分:创建web资源目录,nginx站点配置.

下面来看看如何使用playbook来自动化部署项目到不同环境的多台服务器上:

----

### 准备工作

##### 1.创建主机inventory清单

```
[root@ansible host_vars]$vim /etc/ansible/hosts

[test]
host100 ansible_host=172.16.1.100
host120 ansible_host=172.16.1.120
host102 ansible_host=172.16.1.102
```



##### 1.变量文件

由于不同的服务器所属不同的环境.所以需要为不同的主机定义所属环境的变量.利用ansible的host_vars主机变量可以轻松实现:

* 创建主机变量目录:

```
mkidr /etc/ansible/host_vars
```

* 为每台主机创建变量文件,定义dwd_env变量.这个变量代表了主机所在的环境,

```
[root@ansible host_vars]$vim host100
---
dwd_env: beta

[root@ansible host_vars]$vim host102
---
dwd_env: dev
```

> 一台远程主机属于beta环境,另外一台属于dev环境.
>
> host_vars目录下的文件名和inventory主机清单的主机名要保持一致,只要是主机名文件内的变量都会自动被ansible调用在这台主机下



##### 2.nginx的站点配置文件

这里定义两个站点文件,分别是openapi和internalapi的2套项目环境.配置文件由于需要调用变量,所以使用template模板功能可以很好的满足我们的需求.

* 在playbook目录下创建template目录

```
mkdir /etc/ansible/playbook/template
```

* 创建internalapi的nginx虚拟主机

```
[root@ansible template]$vim nginx-internal.conf 

server {
  listen 80;

#不同的环境的域名也不同.这里调用了dwd_env变量.对于dev环境的主机来说.server_name就是#internal.dev.msf.net

  server_name internal.{{ dwd_env }}.msf.net;
  root /data/apps/msf-internal-api-{{ dwd_env }}/; # 站点根目录

  error_log /data/logs/nginx/msf_internal.error.log;
  access_log /data/logs/nginx/msf_internal.access.log main;


  location / {
    index index.html;
  }
}
```

> 这里为了演示,所以配置文件比较简单.当然结合template的变量,循环,条件判断等功能,也可以根据需求编写出复杂的nginx站点配置文件



* 创建openapi的nginx虚拟主机

```
[root@ansible template]$vim nginx-open.conf 

server {
  listen 80;

  server_name open.{{ dwd_env }}.msf.net;
  root /data/apps/msf-open-api-{{ dwd_env }}/; # 站点根目录

  error_log /data/logs/nginx/msf_open.error.log;
  access_log /data/logs/nginx/msf_open.access.log main;

  # 如果URL中包含app.php，则转发为伪静态格式

  location / {
    index index.html;
  }
}

```



##### 3.  创建nginx的index.html首页文件.这里为了演示也只是编写一个简单文件

```
#创建file目录,用于统一存放Playbook的文件
mkdir /etc/ansible/playbook/files

#创建openapi的首页文件
[root@ansible files]$cat nginx-open.html 

<html>
  <head>
   <title>Welcome to dwd openapi website </title>
  </head>

  <body>
    <h1>nginx,configured by ansible</h1>
    <p>If you see this,Ansible successfully installed and configured nginx .</p>
  </body>
</html>

#创建internalapi的首页文件
[root@ansible files]$cat nginx-internal.html 
<html>
  <head>
   <title>Welcome to dwd internal website </title>
  </head>

  <body>
    <h1>nginx,configured by ansible</h1>
    <p>If you see this,Ansible successfully installed and configured nginx .</p>
  </body>
</html>
```

---

### 编排playbook

nginx所需的配置文件和资源站点文件都准备完毕后,就可以编排playbook了.

playbook大致分为以下几个tasks:

* 检查nginx是否安装,(如果是新服务器,或者在nginx服务器上添加新项目,则步骤可略)
* 添加nginx的yum源,安装nginx (同上)
* 创建nginx站点资源目录
* 在该目录下,创建nginx主页文件
* 创建nginx虚拟主机配置文件(重启nginx服务)
* 启动nginx



下面是具体的Playbook文件

```
[root@ansible playbook]$cat  nginx_env.yml 
---
#all:!主机名的用法表示除了这台服务器外,在所有主机节点执行playbook
 - hosts: all:!host120
   remote_user: root
   tasks:
    
    #这里用到了register注册变量,并且忽略了task执行错误的情况.判断是否有nginx的yum源
     - name: check if there is nginx repo on server
       shell: yum repolist all | grep nginx
       register: nginx_repo
       ignore_errors: True

   #这里用到了when条件判断,如果变量nginx_repo执行失败,则添加nginx的yum源
     - name: install nginx yum repo,if there is no nginx repo
       command: rpm -Uvh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
       when: nginx_repo is failed

     - name: install nginx
       yum: name=nginx state=present
      
   #创建Nginx资源目录,这里使用了嵌套循环.item[0]表示循环'open','internal'.在每次循环item[0]时都循环一遍item[1],也就是'release,vendor,shared.
   #下面的语句类似于shell的命令:
   mkdir -pv /data/apps/msf-{open,internal}-api/{releases,vendor,shared}
   
     - name: create project dir
       file: path=/data/apps/msf-{{ item[0] }}-api-{{ dwd_env }}/{{ item[1] }} recurse=yes owner=root group=root mode=0644 state=directory
       with_nested:
         - ['open','internal']
         - ['releases','vendor','shared']

     - name: create nginx log dir
       file: path=/data/logs/nginx state=directory recurse=yes owner=root group=root mode=0644
    
    #下面使用了with_items的标准循环
     - name: copy nginx conf file
       template: src=template/nginx-{{ item }}.conf dest=/etc/nginx/conf.d/msf-{{ item }}-api.conf
       with_items:
          - open
          - internal        
       notify: restart nginx

     - name: copy nginx web file
       copy: src=files/nginx-{{ item }}.html dest=/data/apps/msf-{{ item }}-api-{{ dwd_env }}/index.html 
       with_items:
         - open
         - internal     

     - name: start nginx
       service: name=nginx state=started

   handlers:
     - name: restart nginx
       service: name=nginx state=restarted

```



执行完毕后,可以在两台节点上检查是否按我们需求创建正确的目录和文件:

```
# 在beta环境服务上,创建了正确的目录结构和首页文件

[root@localhost conf.d]# ll /data/apps/*
/data/apps/msf-internal-api-beta:
total 4
-rw-r--r-- 1 root root 227 Aug  3 21:09 index.html
drw-r--r-- 2 root root   6 Aug  3 21:02 releases
drw-r--r-- 2 root root   6 Aug  3 21:02 shared
drw-r--r-- 2 root root   6 Aug  3 21:02 vendor

/data/apps/msf-open-api-beta:
total 4
-rw-r--r-- 1 root root 226 Aug  3 21:09 index.html
drw-r--r-- 2 root root   6 Aug  3 21:02 releases
drw-r--r-- 2 root root   6 Aug  3 21:02 shared
drw-r--r-- 2 root root   6 Aug  3 21:02 vendor

#创建了正确的beta环境的nginx虚拟主机配置文件
[root@localhost conf.d]# cat msf-open-api.conf 
server {
  listen 80;

  server_name open.beta.msf.net;
  root /data/apps/msf-open-api-beta/; # 站点根目录

  error_log /data/logs/nginx/msf_open.error.log;
  access_log /data/logs/nginx/msf_open.access.log main;

  # 如果URL中包含app.php，则转发为伪静态格式

  location / {
    index index.html;
  }
}

#其他文件检查过程略.
```



在另外一台服务器上检查结果如下:

```
# 创建了正确的dev环境目录:
[root@localhost conf.d]$ll /data/apps/*
/data/apps/msf-internal-api-dev:
total 4
-rw-r--r-- 1 root root 227 Sep  9 03:22 index.html
drw-r--r-- 2 root root   6 Sep  9 03:14 releases
drw-r--r-- 2 root root   6 Sep  9 03:14 shared
drw-r--r-- 2 root root   6 Sep  9 03:14 vendor

/data/apps/msf-open-api-dev:
total 4
-rw-r--r-- 1 root root 226 Sep  9 03:22 index.html
drw-r--r-- 2 root root   6 Sep  9 03:14 releases
drw-r--r-- 2 root root   6 Sep  9 03:14 shared
drw-r--r-- 2 root root   6 Sep  9 03:14 vendor

#创建了正确的nginx配置文件

[root@localhost conf.d]$cat msf-open-api.conf 
server {
  listen 80;

  server_name open.dev.msf.net;
  root /data/apps/msf-open-api-dev/; # 站点根目录

  error_log /data/logs/nginx/msf_open.error.log;
  access_log /data/logs/nginx/msf_open.access.log main;

  # 如果URL中包含app.php，则转发为伪静态格式

  location / {
    index index.html;
  }
}

```

可见所有的目录和配置文件都按我们需求做到自动化部署,这为我们节省了大量的时间,工作效率大幅提升.而且也能很轻松的胜任更复杂的项目环境配置工作