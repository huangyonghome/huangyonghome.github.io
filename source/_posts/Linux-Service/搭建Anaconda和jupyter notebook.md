---
title:  搭建Anaconda和jupyter notebook
date: 2020-08-25 22:59:58
tags:  Anaconda
categories: Linux-Service
comments: true
copyright: true
---



## 搭建Anaconda和jupyter notebook

###  一.什么是Anaconda

Anaconda可以便捷获取包且对包能够进行管理，同时对环境可以统一管理的发行版本。Anaconda包含了conda、Python在内的超过180个科学包及其依赖项。

## 2. 特点

Anaconda具有如下特点：

- 开源
- 安装过程简单
- 高性能使用Python和R语言
- 免费的社区支持

其特点的实现主要基于Anaconda拥有的：

- conda包
- 环境管理器
- 1,000+开源库

<!--more-->

---

###  3.Anaconda安装

[Anaconda官方](https://www.anaconda.com/)提供了三种不同的版本,除了Individual Edition个人版免费以外,其他2种版本都是收费的.所以这里选择Anaconda个人版.

#### 3.1 Docker安装(有坑,弃用了)

我刚开始选择的是用Docker运行,使用的是Anaconda的镜像:[continuumio/anaconda3](https://hub.docker.com/r/continuumio/anaconda3).用Dockerfile在此基础之上安装了多个python扩展模块自定义了一个镜像.Dockerfile内容如下:

```
FROM continuumio/anaconda3:latest
LABEL author=jessehuang
LABEL description="自定义制作annaconda镜像"

#更新debian源.使用清华大学的debian 10 brust的源
RUN apt-get update && apt-get install -y apt-transport-https ca-certificates

ADD sources.list /etc/apt/sources.list

#安装python模块的依赖.否则thriftpy的安装会出现问题

RUN apt-get update && apt-get install -y --no-install-recommends \
    python-dev \
    vim \
    gcc \
    g++ \
    libsasl2-dev 

#安装python扩展模块

ADD requirement.txt /tmp/requirement.txt 
RUN pip install -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com -r /tmp/requirement.txt

RUN /opt/conda/bin/conda install jupyter -y --quiet && mkdir /opt/notebooks 
EXPOSE 8888

#启动命令
CMD ["/opt/conda/bin/jupyter","notebook", "--notebook-dir=/opt/notebooks","--ip='*'","--port=8888","--no-browser","--allow-root"]
```

requirement.txt内容如下:

```
pyecharts
pymysql
psycopg2-binary
six
bit_array
thriftpy
thrift_sasl==0.2.1
impyla
jupyter_contrib_nbextensions
```

docker部署的anaconda在运行jupyter notebook的时候提示`kernel restarting`.容器日志内容如下:

```
[I 07:25:59.512 NotebookApp] Restoring connection for da9ccce4-dac5-42d1-aef3-2f37c0689e0c:61003e8cadbc408b9ee98174729d5086
[I 07:25:59.541 NotebookApp] Starting buffering for da9ccce4-dac5-42d1-aef3-2f37c0689e0c:61003e8cadbc408b9ee98174729d5086
[I 07:25:59.565 NotebookApp] Restoring connection for da9ccce4-dac5-42d1-aef3-2f37c0689e0c:61003e8cadbc408b9ee98174729d5086
[I 07:25:59.595 NotebookApp] Starting buffering for da9ccce4-dac5-42d1-aef3-2f37c0689e0c:61003e8cadbc408b9ee98174729d5086
[I 07:25:59.620 NotebookApp] Restoring connection for da9ccce4-dac5-42d1-aef3-2f37c0689e0c:61003e8cadbc408b9ee98174729d5086
[W 07:26:04.401 NotebookApp] Replacing stale connection: 63b3942c-a225-4692-8989-893af2c992cc:743009a3e3684e64a9b9fcceedc30775
[W 07:26:05.062 NotebookApp] Replacing stale connection: a9543d90-08b5-4395-8a3c-f7bbec9f44a1:0bc5b6496d59406b82ab54a181db7215
```

这个故障在搜遍了Google和baidu后也无解,尝试了网上各种解决办法也不行.所以果断弃掉,转而使用Linux虚拟机部署

### 3.2 Linux部署安装(成功)

Anaconda的[安装官方文档](https://docs.anaconda.com/anaconda/install/linux/).建议已root用户安装,或者你的普通用户有执行`/home/$(USER)/anaconda3/bin/python3`的sudo权限

1.首先下载python的anaconda安装脚本,建议选择python3的版本

https://www.anaconda.com/products/individual#linux

2.安装依赖包

```
yum install libXcomposite libXcursor libXi libXtst libXrandr alsa-lib mesa-libEGL libXdamage mesa-libGL libXScrnSaver
```

3.执行下载下来的脚本文件.根据提示,一直选择默认就可以了,官方不建议更改安装路径

```
bash ~/Anaconda3-2020.02-Linux-x86_64.sh
```

>We recommend you accept the default install location. Do not choose the path as /usr for the Anaconda/Miniconda installation.

4.如果提示`Thank you for installing Anaconda<2 or 3>!`则说明安装完成,应用环境变量

```
source ~/.bashrc
```

5.验证安装结果

```
condal list #显示已经安装的包和版本好
```

至此,Anaconda和anaconda自带的jupyter notebook都已经安装完了

---

### 4.jupyter notebook 扩展模块安装

公司的BI团队需要使用jupyter notebook访问MySQL、pgsql、hive,画图等工作,所以需要安装python的部分扩展模块.首先查看服务器上pip版本.需要使用python3的pip来安装模块

```
#显示pip版本是python3
(base) [root@anaconda ~]# pip --version
pip 20.0.2 from /root/anaconda3/lib/python3.7/site-packages/pip (python 3.7)
(base) [root@anaconda ~]#
```

1. 安装依赖文件,否则安装`thriftpy`模块会报错

```
yum install gcc-c++ gcc python3-devel python-dev cyrus-sasl cyrus-sasl-devel cyrus-sasl-lib
```

> 如果是ubuntu系统,则需要安装如下安装包: apt-get install -y python-dev gcc g++ libsasl2-dev

2. 编写requirement文件,将所需要安装的python扩展模块加入到文件中

```
pyecharts
pymysql
psycopg2-binary
six
bit_array
thriftpy
thrift_sasl==0.2.1
impyla
jupyter_contrib_nbextensions
```

3.安装

```
pip install -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com -r requirement.txt
```

安装完依赖包`jupyter_contrib_nbextensions`以后,安装一个服务:

```
jupyter-contrib-nbextension install --user
```

4.初始化jupyter配置文件,会在默认路径下初始化一个配置文件`/root/.jupyter/jupyter_notebook_config.py`

```
jupyter notebook --generate-config
```

5.编辑文件,修改如下配置

```
c.NotebookApp.ip='当前服务器IP'
c.NotebookApp.notebook_dir = u'/data/notebooks' #jupyter笔记的工作目录
c.NotebookApp.open_browser = False #服务端无需启动浏览器
c.NotebookApp.port = 80 #监听端口,默认是8888
c.NotebookApp.allow_root = True  #允许root用户运行jupyter notebook
c.NotebookApp.allow_origin = '*' #允许所有来源访问,解决跨域问题
c.NotebookApp.quit_button = False #关闭jupyter notebook浏览器界面的quit按钮功能.因为quit按钮会关闭jupyter服务
```

6.启动服务.记下pid号,后面要重启

```
nohup jupyter-notebook --config=/root/.jupyter/jupyter_notebook_config.py &
```

7.打开浏览器,输入IP地址,此时会进入jupyter的界面.要求输入Token,或者使用Token设置一个密码

![image-20200708105619499](https://img2.jesse.top/image-20200708105619499.png)



8.根据提示,使用命令`jupyter notebook list`查看Token

```
base) root@anaconda:/# jupyter notebook list
Currently running servers:
http://localhost:8888/?token=5469d940ce3a70950299e5c907e1d47b09acc61457348cd9 :: /data/notebooks
```

9.拿到Token后,在浏览器中下方位置设置一个新密码.

![image-20200708105756748](https://img2.jesse.top/image-20200708105756748.png)

10.设置完密码后,成功登陆了jupyter的工作界面.此时kill掉jupyter进程,重启

```
#关闭进程
kill $PID

#重新启动
nohup jupyter-notebook --config=/root/.jupyter/jupyter_notebook_config.py &
```

11.此时再次打开浏览器,就提示输入密码

![image-20200708110014017](https://img2.jesse.top/image-20200708110014017.png)

12.输入密码后,成功登陆

![image-20200708110048546](https://img2.jesse.top/image-20200708110048546.png)

