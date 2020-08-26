---
title: docker学习笔记---Dockerfile
date: 2020-06-29 11:59:58
tags:  docker
categories: docker
comments: true
copyright: true
---



### Dockerfile

---

### 介绍

到目前位置我们接触的镜像都是从docker官方hub.docker.com所下载的官方镜像.官方镜像一般分为两类:

* OS镜像
* 应用镜像.(例如nginx.php-fpm,mysql)

但是无论是哪种镜像,官方的标准镜像(或者是通用镜像)都只包含最基础的系统或者应用环境,如果需要使用适合个人或者企业场景的自定义镜像,那么就只能自己制作镜像.

**制作镜像的方法:**

* 以容器为基础,制作镜像

这有点类似于快照,用一个基础镜像运行容器,然后在容器的交互式界面,安装或者配置之定义环境,然后将该容器制作成一个镜像.但是这种方法有很大的缺陷.

例如,如果有多个环境的容器,那么可能容器的应用配置文件不一样,此时可能需要将多个容器制作成多个镜像.例如dev环境容器,beta环境容器,生产环境容器等.而且容器一旦修改以后,可能又需要重新commit制作成镜像,这样一来,镜像的体积将无限大.

* Dockerfile制作镜像

Dockerfile可以用来编译一个docker镜像.Dockerfile是一个包含一系列指令的文本文档,非常类似于一个shell脚本,.docker调用dockerfile中每行的命令,聚集成最终的镜像.使用`docker build`命令,用户可以依据dockerfile和上下文编译一个镜像.

---

### Dockerfile语法格式

dockerfile的格式非常简单.只有2种:

* `#`开头的语句表示注释
* `INSTRUCTIONS arguments` 指令 参数

`instructions`指令大小写非敏感,但是约定俗成的写法是指令全部用大写字母,dockerfile按自上而下的顺序,一次执行每一个指令.

> Dockerfile文件中第一个指令必须是`FROM`指令,用来表示当前的docker镜像是哪个基础镜像.在dockerfile中,必须要一个基础镜像.(有点类似编程中,所有的类必须要有一个基类)

---

###  Dockerfile上下文

docker build编译镜像时,默认在当前目录下寻找文件名为`Dockerfile`的文件.如果找不到,则会报错,所以Dockerfile(第一个字母必须大写)的文件名一般是固定的.

docker会将当前目录下的Dockerfile和所有文件打包添加发送到docker daemon服务端.所以一般情况下创建一个空目录编辑Dockerfile文件.然后将需要放到镜像中的文件放进和dockerfile同一目录下,或者子目录下.当然,也可以像Git那样,在当前目录下编辑一个名为`.dockeringnore`的隐藏文件,列出排除文件或者目录.这样dockerfile在制作镜像时,不会将该文件或者目录提交到docker服务端

> 为什么需要一个空目录防止Dockerfile文件呢?
>
> 因为docker在编译镜像之前,就会将该目录下的所有文件打包发送到docker daemon服务端中,并且加载到镜像.所以最终产生的镜像体积会非常庞大.(无论你是否使用了目录下的文件或者文件夹).试想一下如果在`/`目录下打包Dockerfile镜像,会是什么后果



由于上下文的关系,在Dockerfile中添加文件到docker镜像时.不要使用绝对路径.例如/home/work/a.txt.这是因为docker deamon只能识别到当前上下文环境,无法识别到其他目录.但是可以使用当前上下文的相对路径.

---

### Dockerfile构建过程

使用`docker build`命令可以将当前目录下的Dockerfile文件编译成一个镜像,docker daemon会隐含式的用Dockerfile中`FROM`指令指定的镜像启动一个容器,并且从上到下依次在该容器中执行Dockerfile文件中的指令.最终将改容器打包成一个新的镜像.

其本质上和我们启动一个容器,执行命令,然后保存成一个镜像没有多大区别,只不过是docker的`build`命令帮助我们完成以上一系列动作.所以需要注意的是,我们在Dockerfile中编辑的指令和参数,其作用对象和执行环境并非是宿主机,而是基础镜像..如果基础镜像不支持Dockerfile文件中的指令,那么`build`执行过程中很可能会报错

---

###  Dockerfile变量

Dockerfile支持变量替换,其变量赋值和调用方法和shell脚本中非常类似.

* 变量赋值: ENV version 3.0 #设一个名为version的变量,其值为3.0

* 变量调用: \$version 或者 \${version}

> 当然,还支持`${variable: -word}`和`${variable: +word}`等更多的变量字符串截取.这里不做说明

变量可以方便我们构建不同版本的应用.在编译多个版本的同一应用镜像时,我们不需要去修改Dockerfile的指令内容,只需要修改变量名即可

---

### dockerfile打包镜像命令

命令`docker build`用来编译制作镜像.其支持许多参数,但是常用的格式如下:

```
docker build -t image:tag .
```

使用`-t`参数指定新的镜像和标签, `.`小数点表示当前目录.docker会在当前目录下搜索Dockerfile文件,且将当前目录的所有文件和子目录打包到docker daemon

---

使用dockerfile需要注意一些事项

**1.上下文**

docker build编译镜像时,会将当前目录下的Dockerfile和所有文件打包添加发送到docker daemon服务端.所以一般情况下创建一个空目录编辑dockerfile文件.然后将需要copy和add的文件放进和dockerfile同一目录下.

dockerfile中的`copy`以及`add`命令,添加文件到docker镜像中时.不要使用绝对路径.例如/home/work/a.txt..docker deamon只能识别到当前上下文环境,无法识别到其他目录.但是可以使用当前上下文的相对路径.

**2.分层**

dockerfile编译镜像时,每条指令都是一个镜像层.除了From指令外,每一行指令都是基于上一行生成的临时镜像运行一个容器.执行一条指令就类似于docker commit命令生成一个新的镜像.所以两条指令之间互不关联.

<!--more-->

例如,下列的dockerfile并不能在/data/目录下创建files文件.

```
FROM ubuntu
RUN mkdir /data
RUN cd /data
RUN touch files
```

下列的dockerfile甚至不会创建/data/file文件,也不会修改/data/目录权限
```
FROM ubuntu
RUN useradd foo
VOLUME /data
RUN touch /data/file
RUN chown -R foo:foo /data
```

想要实现这个需求,可以这样写:

```
FROM ubuntu
RUN useradd foo
RUN mkdir /data && touch /data/file
RUN chown -R foo:foo /data
VOLUME /data
```
---

**3.精简**

由于dockerfile在构建镜像时,dockerfile文本中每一行语句会产生每一层镜像.

例如下面这个dockerfile:

```
From ubuntu
RUN apt-get update
RUN apt-get -y install vim git wget net-tools
RUN useradd foo
RUN mkdir /data
RUN touch /data/file
RUN chown -R foo:foo /data
```
在编译时,每一个RUN语句都会构建一层镜像.(实际上所有指令都是这样,不仅仅是RUN)

```
[root@localhost test]$docker build -t test:v1 .
Sending build context to Docker daemon  2.048kB
Step 1/7 : From ubuntu
 ---> 94e814e2efa8
Step 2/7 : RUN apt-get update
 ---> Using cache
 ---> 5520126e7fcc
Step 3/7 : RUN apt-get -y install vim git wget net-tools
 ---> Using cache
 ---> cb24e170539c
Step 4/7 : RUN useradd foo
 ---> Using cache
 ---> ca31aeba0309
Step 5/7 : RUN mkdir /data
 ---> Using cache
 ---> d5c6e0f32f6b
Step 6/7 : RUN touch /data/file
 ---> Using cache
 ---> 9c4b06e9b25d
Step 7/7 : RUN chown -R foo:foo /data
 ---> Using cache
 ---> 75ecea0b0795
Successfully built 75ecea0b0795
Successfully tagged test:v1
```

这种写法会导致镜像层非常多,镜像文件也会相对较大.所以一般推荐更精简的语法,每一条功能相同的语句,尽量写在一行.上面的dockerfile可以优化成:

```
From ubuntu
RUN apt-get update \
    && apt-get -y install \
       vim \
       git \
       wget \
       net-tools

RUN useradd foo
RUN mkdir /data &&  touch /data/file &&  chown -R foo:foo /data
```
这次编译只需构建4层镜像

```
[root@localhost test]$docker build -t test:v1 .
Sending build context to Docker daemon  2.048kB
Step 1/4 : From ubuntu
 ---> 94e814e2efa8
Step 2/4 : RUN apt-get update     && apt-get -y install        vim        git        wget        net-tools
 ---> Using cache
 ---> 7aa2bc9041e0
Step 3/4 : RUN useradd foo
 ---> Using cache
 ---> 5a13764414e6
Step 4/4 : RUN mkdir /data &&  touch /data/file &&  chown -R foo:foo /data
 ---> Using cache
 ---> bd61817d7526
Successfully built bd61817d7526
Successfully tagged test:v1
```

4.使用`no-install-recommends`

如果是使用APT包管理器,则应该在执行apt-get install 命令时加上`no-install-recommends``参数.这样ATP就仅安装核心依赖.而不安装其他推荐和建议的包,这会显著减少不必要包的下载数量

---

### Dockerfile指令介绍

介绍完Dockerfile的概念和特点后,接下来了解一下Dockerfile语法中的具体指令的介绍和用法


下面是一个例子:

```
# Use an official Python runtime as a parent image
FROM python:2.7-slim

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
COPY . /app

# Install any needed packages specified in requirements.txt
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# Make port 80 available to the world outside this container
EXPOSE 80

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]
```

---

* FROM 

格式: `FROM image` 或者 `FROM image:tag`

表示从一个基础镜像构建.Dockerfile必须以FROM语句作为第一条非注释语句.默认情况下docker回在当前主机本地查找该镜像,如果镜像不存在,则默认会在docker hub官方拉取镜像文件.或者从指定的registry中寻找,下载镜像

具体格式如下

```
FROM image      #指定镜像名,不加tag就默认是latest镜像
FROM image:tag 
FROM <registry><repository>:tag #指定具体镜像注册中心的镜像,非docker官方hub站点镜像
```

* MAINTAINER(deprecated,已废弃)

用来表示Dockerfile的制作者的信息.是一个可选指令.一般都会附加到Dockerfile中,表面制作镜像的人员,或者联系方式.例如:

```
MAINTAINER jesse
Maintainer huangyong@doweidu.com
```

这个命令已经被废弃,取而代之的是LABEL指令.但是MAINTAINER指令依然被新版本的docker兼容

* LABEL

表示为镜像添加元数据.kv格式的指令

```
LABEL <key>=<value> <key>=<value>
LABEL author=jesse mail=huangyong@doweidu.com
```

* WORKDIR: 表示工作目录,后续的相对路径也是基于这个目录.容器启动以后,使用`exec`命令交互式界面接入容器,也是现实这个目录

* COPY

格式: `copy src dest`或者`COPY ["src"....."dest"]`

复制宿主机上的文件到镜像中.src是当前上下文中的文件或者目录.dest是容器中的目标文件或者目录.

src指定的源可以有多个.此外 src还支持通配符.例如: `COPY hom* /mydir/` 表示添加所有当前目录下的hom开头的文件到目录/mydir/下

<dest>可以是文件或者目录.但是必须是镜像中的绝对路径,或者是WORKDIR的相对路径.若<dest>以反斜杠/结尾,则指向的是目录,否则指向文件.

当 src 有多个源时, dest必须是目录.如果 dest 目录不存在,则会自动被创建

如果目录当中含有空白字符,为了避免歧义,则应该使用第二种格式

如果src是目录,则其内部文件或者子目录会被递归复制,但是src目录本身并不会复制.举个例子:

```
#shell中命令
cp -r a b/ #表示将a目录复制到b目录下,最终b目录是: b/a/.....

#dockerfile COPY指令
COPY a b/ #表示将a目录下的所有内容复制到b目录.相当于shell中的cp -r a/* b/ .最终b目录是: b/..... 
```



* ADD

格式: `ADD src dest`

 ADD和COPY命令有相同功能,指令用法也基本一样.都支持复制本地文件到镜像里.但ADD能从互联网的URL下载文件到镜像..src还可以是一个本地的压缩归档文件.ADD会自动将tar,gz等压缩包上传到镜像后进行解压.

 但是如果src是一个URL的归档格式文件,则不会自动解压.



* VOLUME

`VOLUME`用于在镜像中创建一个挂载点目录(docker自行管理的挂载卷),以挂载Docker宿主机上的卷或者其他容器的卷.

用法:

```
VOLUME mountpoint
VOLUME ["mountpoint"]
```

`volume`会将容器内的目录自行挂载到docker宿主机上,而无需在启动容器时候使用`-v`参数挂载.

> 但是无法使用指定挂载卷,也就是无法指定宿主机的目录挂载,只能使用docker自行管理的挂载卷




* RUN

RUN命令有两种格式:

```
RUN <command> (shell格式)  
RUN ["executable","param1","param2"] (exec格式)
```

RUN指令的两种格式表示命令在容器中的两种运行方式.当使用shell格式时,命令通过` /bin/sh -c `运行.当使用exec格式时.命令直接运行,不调用shell程序.exec格式中的参数会被当成JSON数组被Docker解析.所以必须使用双引号,不能使用单引号. 

另外由于exec格式不会在shell中运行.所以无法识别ENV环境变量.例如当执行`CMD ["echo","$HOME"]`时,$HOME不会被变量替换.如果希望运行shell程序.可以写成

```
CMD ["sh","-c","echo","$HOME"]
```

> RUN命令和CMD命令都是表示运行一个命令,但是他们运行的阶段不一样,RUN命令是在build镜像时候运行指令,CMD是在build完成镜像后,启动容器运行的命令




* EXPOSE: 镜像需要暴露出来的端口. 

> 要注意的是,这里只是说明镜像需要暴露哪些端口,在镜像构建完毕,启动容器时,仍然需要-p参数来映射端口,否则端口不会自动映射

用法:

```
EXPOSE port/protocol port/protocol #protocol可以不指定,默认是TCP协议,可以在一行中暴露多个端口
```

> EXPOSE无法指定宿主机的IP或者端口,因为镜像运行的宿主机是非固定的,并不知晓宿主机的环境.



* ENV

  格式: `ENV <key> <value>` 或者 `ENV <key>=<value> <key>=<value>`
  
  ENV指令用来声明环境变量,并且可以被(ADD,COPY,WORKDIR等)指令调用
  
  Dockerfile变量赋值和调用方法和shell脚本中非常类似.
  
  * 变量赋值: ENV version 3.0 #设一个名为version的变量,其值为3.0
  
  * 变量调用: \$version 或者 \${version}
  
  > 当然,还支持`${variable: -word}`和`${variable: +word}`等更多的变量字符串截取.这里不做说明
  
  变量可以方便我们构建不同版本的应用.在编译多个版本的同一应用镜像时,我们不需要去修改Dockerfile的指令内容,只需要修改变量名即可
  
  EVN像镜像传递环境变量,不仅仅是影响到镜像本身,而且一旦镜像启动为容器时,还会注入到容器内部.在容器中可以继续使用该变量.如果在容器启动时候,覆盖了同名的环境变量,则容器中以覆盖后的环境变量值为准,但是不会更改镜像中已存在的事实结果.
  
  举个例子:
  
  ```
  #dockerfile中设置环境变量
  ENV NGINX_VERSION 1.17
  ADD http://nginx.org/download/nginx-1.17.tar.gz /usr/local/src/
  
  #生成镜像,启动容器,覆盖同名环境变量
  docker run -e NGINX_VERSION 1.16 
  
  此时再容器中NGINX_VERSION的环境变量值为1.16.但是/usr/local/src/目录下的gz文件仍然是nginx 1.17
  ```
  
  
  
* CMD 

CMD命令有3种格式:

```
CMD <command> (shell格式)  
CMD ["executable","param1","param2"] (exec格式)  
CMD ["param1","param2"] (为ENTRYPOINT命令提供参数)
```

CMD提供容器启动后执行的命令.或者是为ENTRYPOINT传递一些参数.一个dockerfile文件只允许存在一条CMD指令.如果存在多条CMD指令,以最后一条为准.但是如果用户在 docker run 时指定了命令,则会覆盖CMD中的指令



* ENTRYPOINT

ENTRYPOINT有两种格式.和上文CMD一样分为shell格式和exec格式.

ENTRYPOINT和CMD类似,指定容器启动时执行的命令.和CMD一样一个Dockerfile文件中可以有多个ENTRYPOINT命令.但只有最后一条生效.但是又有一些区别.当使用shell格式时,ENTRYPOINT会忽略任何CMD指令和 `docker run`启动容器时手动输入的指令.并且会运行在 /bin/sh -c环境中,成为它的子进程.进程在容器中PID不是1,也不能接收UNIX信号.(也就是在执行 `docker stop <container>`时,进程接收不到SIGTERM指令)

当使用exec格式时, docker run 手动指定的命令,将作为参数覆盖CMD指定的参数传递到ENTRYPOINT.(也就是说 docker run启动容器时指定的不再是具体命令,而是命令的参数).

---

创建上面dockerfile中所需要的app.py和requirements.txt文件,并且将他们和Dockerfile文件放在同一目录下:

requirements.txt:

```
[root@localhost docker_python]$cat requirements.txt
Flask
Redis

```
app.py:

```
root@localhost docker_python]$cat app.py
from flask import Flask
from redis import Redis, RedisError
import os
import socket

# Connect to Redis
redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)

app = Flask(__name__)

@app.route("/")
def hello():
    try:
        visits = redis.incr("counter")
    except RedisError:
        visits = "<i>cannot connect to Redis, counter disabled</i>"

    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>" \
           "<b>Visits:</b> {visits}"
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname(), visits=visits)

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
[root@localhost docker_python]$
```
---

#### 开始构建镜像

首先确保Dockerfile里所需的文件,以及Dockerfile都在同一目录下:

```
[root@localhost docker_python]$ls
app.py  Dockerfile  requirements.txt
```
运行以下命令来构建一个镜像.使用--tag参数(或者-t),可以为镜像打个标签:

```
docker build --tag=friendlyhello .
```

构建过程略..在构建过程中,注意以下现象:

1.构建镜像层:

部分指令会创建一个新的镜像层,而有些指令则不会.关于如何区分命令是否会新建镜像层,一个基本的原则是:

如果指令的作用是像镜像中添加新的文件或者程序,那么就会新建镜像层.(例如:RUN,COPY,ADD,FROM等)

如果只是告诉docker如何构建或者运行应用程序,增加或者修改容器的元数据,那么不会构建新的镜像层.(例如:WORKDIR,EXPOSE,ENV,ENTERPOINT等)

2.构建步骤:

基本等过程大致为:

运行临时容器---->在该容器中运行Dockerfile指令---->将运行结果保存为一个新等镜像层------> 删除临时容器

构建完成后,通过以下命令可以看到构建的镜像

```
[root@localhost docker_python]$docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED              SIZE
friendlyhello              latest              f091d1bb803c        About a minute ago   131MB

```
> 或者也可以是输入以下命令 docker image ls

> 可能会疑惑,为什么tag标签是latest..镜像的完整标签格式应该是:friendlyhello:lastest.
如果需要在构建镜像时指定版本.可以使用: --tag=friendlyhello:v0.0.1

----

#### 使用构建的镜像启动一个容器

输入以下命令,利用刚才的镜像启动一个容器:

```
docker run -p 4000:80 friendlyhello
```
执行结果:

```
[root@localhost docker_python]$docker run -p 4000:80 friendlyhello
 * Serving Flask app "app" (lazy loading)
 * Environment: production
   WARNING: Do not use the development server in a production environment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
```
 * docker run 表示启动一个容器
 * -p 宿主机端口:容器端口  表示将宿主机的端口映射给容器.如果是-P 80 表示随机映射一个宿主机的端口给容器


此时可以在其他电脑上访问这个容器的80端口,下面是在我的PC上访问宿主机的4000端口,也就是刚才启动的容器

```
 ✘ huangyong@huangyong-Macbook-Pro  ~  curl http://10.0.0.50:4000
<h3>Hello World!</h3><b>Hostname:</b> f9b1b804404f<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>%
```

容器默认是在前台执行,加上-d参数可以时容器运行在后台:

```
docker run -d -p 4000:80 friendlyhello
```

docker ps命令可以显示正在运行中的容器

```
[root@localhost docker_python]$docker ps
CONTAINER ID        IMAGE                      COMMAND             CREATED             STATUS              PORTS                    NAMES
fcf7d29ac627        friendlyhello              "python app.py"     5 seconds ago       Up 1 second         0.0.0.0:4000->80/tcp     stoic_colden
```
> docker container ls命令也有同样的效果

---

这一节(包括第3小节)涉及到的基础命令如下:

```
docker build -t friendlyhello .  # Create image using this directory's Dockerfile
docker run -p 4000:80 friendlyhello  # Run "friendlyname" mapping port 4000 to 80
docker run -d -p 4000:80 friendlyhello         # Same thing, but in detached mode
docker container ls                                # List all running containers
docker container ls -a             # List all containers, even those not running
docker container stop <hash>           # Gracefully stop the specified container
docker container kill <hash>         # Force shutdown of the specified container
docker container rm <hash>        # Remove specified container from this machine
docker container rm $(docker container ls -a -q)         # Remove all containers
docker image ls -a                             # List all images on this machine
docker image rm <image id>            # Remove specified image from this machine
docker image rm $(docker image ls -a -q)   # Remove all images from this machine
docker login             # Log in this CLI session using your Docker credentials
docker tag <image> username/repository:tag  # Tag <image> for upload to registry
docker push username/repository:tag            # Upload tagged image to registry
docker run username/repository:tag                   # Run image from a registry
```