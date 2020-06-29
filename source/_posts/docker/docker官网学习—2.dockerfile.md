---
title: docker学习笔记---Dockerfile
date: 2020-06-29 11:59:58
tags:  docker
categories: docker
comments: true
copyright: true
---



### Dockerfile

Dockerfile可以用来编译一个docker镜像.Dockerfile是一个包含一系列指令的文本文档,使用docker build命令,用户可以依据dockerfile和上下文编译一个镜像.

使用dockerfile需要注意一些事项

**1.上下文**

docker build编译镜像时,会将当前目录下的Dockerfile和所有文件打包添加发送到docker daemon服务端.所以一般情况下创建一个空目录编辑dockerfile文件.然后将需要copy和add的文件放进和dockerfile同一目录下.

dockerfile中的copy以及add命令,添加文件到docker镜像中时.不要使用绝对路径.例如/home/work/a.txt..docker deamon只能识别到当前上下文环境,无法识别到其他目录.但是可以使用当前上下文的相对路径.

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

4.使用no-install-recommends

如果是使用APT包管理器,则应该在执行apt-get install 命令时加上no-install-recommends参数.这样ATP就仅安装核心依赖.而不安装其他推荐和建议的包,这会显著减少不必要包的下载数量

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

格式: FROM image 或者 FROM image:tag

表示从一个基础镜像构建.Dockerfile必须以FROM语句作为第一条非注释语句.

* WORKDIR: 表示工作目录,后续的相对路径也是基于这个目录

* COPY

格式: copy src dest

复制宿主机上的文件到镜像中.src是当前上下文中的文件或者目录.dest是容器中的目标文件或者目录.src指定的源可以有多个.此外 src还支持通配符.例如: COPY hom* /mydir/ 表示添加所有当前目录下的hom开头的文件到目录/mydir/下

<dest>可以是文件或者目录.但是必须是镜像中的绝对路径,或者是WORKDIR的相对路径.若<dest>以反斜杠/结尾,则指向的是目录,否则指向文件.当 src 有多个源时, dest必须是目录.如果 dest 目录不存在,则会自动被创建

* ADD

格式: ADD src dest

 ADD和COPY命令有相同功能,都支持复制本地文件到镜像里.但ADD能从互联网的URL下载文件到镜像..src还可以是一个本地的压缩归档文件.ADD会自动将tar,gz等压缩包上传到镜像后进行解压.

 但是如果src是一个URL的归档格式文件,则不会自动解压.


* RUN

RUN命令有两种格式:

RUN <command> (shell格式)  
RUN ["executable","param1","param2"] (exec格式)

RUN指令的两种格式表示命令在容器中的两种运行方式.当使用shell格式时,命令通过 /bin/sh -c 运行.当使用exec格式时.命令直接运行,不调用shell程序.exec格式中的参数会被当成JSON数组被Docker解析.所以必须使用双引号,不能使用单引号. 

另外由于exec格式不会在shell中运行.所以无法识别ENV环境变量.例如当执行CMD ["echo","$HOME"]时,$HOME不会被变量替换.如果希望运行shell程序.可以写成

```
CMD ["sh","-c","echo","$HOME"]
```


* EXPOSE: 镜像需要暴露出来的端口. 

> 要注意的是,这里只是说明镜像需要暴露哪些端口,在镜像构建完毕,启动容器时,仍然需要-p参数来映射端口,否则端口不会自动映射

* ENV

  格式: ENV <key> <value> 或者 ENV <key>=<value>
  
  ENV指令用来声明环境变量,并且可以被(ADD,COPY,WORKDIR等)指令调用.调用ENV环境变量的格式和shell一样:\$variable_name或者 \${variable_name}
  
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

ENTRYPOINT和CMD类似,指定容器启动时执行的命令.和CMD一样一个Dockerfile文件中可以有多个ENTRYPOINT命令.但只有最后一条生效.但是又有一些区别.当使用shell格式时,ENTRYPOINT会忽略任何CMD指令和 docker run启动容器时手动输入的指令.并且会运行在 /bin/sh -c环境中,成为它的子进程.进程在容器中PID不是1,也不能接收UNIX信号.(也就是在执行 docker stop <container>时,进程接收不到SIGTERM指令)

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