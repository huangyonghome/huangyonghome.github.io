---
title:  24.Python之路 - tarfile&zipfile&shutil模块
date: 2020-06-26 09:20:58
tags:  python
categories: [python,03-Modules]
comments: true
copyright: true
---



# Python之路 -  tarfile&zipfile&shutil模块

##  介绍 🍀

tarfile和zipfile,shutil都是文件压缩,打包,解压的模块.其中shuttile还支持文件的拷贝,复制等功能

##  zipfile模块 🍀


* 单个文件压缩
  
```
import zipfile
with zipfile.ZipFile('log.zip','w') as z:
    z.write('sys.log')     #将文件sys.log添加到log.zip这个压缩包.下面2行代码功能一样
    z.write('test.log')
    z.write('advance.log')

```
* 查看上面log.zip压缩包内的文件

<!--more-->

```
#用```r```只读模式打开压缩包
with zipfile.ZipFile('log.zip','r') as z:
    print(z.namelist())  #namelist方法以列表形式返回压缩包内的所有文件

>>> ['sys.log', 'test.log', 'advance.log']
```

* 追加新的文件到log.zip压缩包

```
#用```a```追加模式打开压缩包
with zipfile.ZipFile('log.zip','a') as z:
    z.write('user.db')
    #查看文件
    print(z.namelist())

>>> ['sys.log', 'test.log', 'advance.log', 'user.db']

```

* 解压压缩包.解压的目标目录如果不存在,会自动创建

```
with zipfile.ZipFile('log.zip','r') as z:
    z.extractall(path='log') #解压到log目录下

#查看log目录下文件:
print(os.listdir('log'))

>>> ['advance.log', 'sys.log', 'test.log', 'user.db']
```

* 压缩目录下的所有文件

```
def addfile(zipfilename, dirname):
    if os.path.isfile(dirname):
        with zipfile.ZipFile(zipfilename, 'a') as z:
            z.write(dirname)
    else:
        with zipfile.ZipFile(zipfilename, 'a') as z:
            for root, dirs, files in os.walk(dirname): 获取子目录下的所有文件以及父目录名
                for single_file in files:
                    filepath = os.path.join(root, single_file) 将子目录和子目录下文件名路径拼接
                    z.write(filepath)

addfile('module2-1.zip', 'module2')
```

## tarfile模块  🍀

tarfile模块和zipfile模块使用方法大同小异.

* 压缩单个文件
* 
```
with tarfile.open('module2.tar','w') as tar:
    tar.add('sys.log')
    tar.add('test.log')
```

* 解压文件

```
with tarfile.open('a.tar', 'r') as tar:
    print(tar.getmembers())     # 查看压缩包内文件成员
    # tar.extract('test.txt')  # 可选择解压某个文件
    # tar.extractall('ccc')  # 可设置解压路径
    tar.extractall()  # 解压全部
```

* 压缩目录下的所有文件

```
def compress_file(tarfilename, dirname):    # tarfilename是压缩包名字，dirname是要打包的目录
    if os.path.isfile(dirname):
        with tarfile.open(tarfilename, 'w') as tar:
            tar.add(dirname)
    else:
        with tarfile.open(tarfilename, 'w') as tar:
            for root, dirs, files in os.walk(dirname):
                for single_file in files:
                    # if single_file != tarfilename:
                    filepath = os.path.join(root, single_file)
                    tar.add(filepath)

compress_file('test.tar', 'test.txt')
compress_file('t.tar', '.')
```

* 添加文件到已有的压缩包中

```
def addfile(tarfilename, dirname):    # tarfilename是压缩包名字，dirname是要打包的目录
    if os.path.isfile(dirname):
        with tarfile.open(tarfilename, 'a') as tar:
            tar.add(dirname)
    else:
        with tarfile.open(tarfilename, 'a') as tar:
            for root, dirs, files in os.walk(dirname):
                for single_file in files:
                    # if single_file != tarfilename:
                    filepath = os.path.join(root, single_file)
                    tar.add(filepath)

addfile('t.tar', 'ttt.txt')
addfile('t.tar', 'ttt')
```

## shutil模块  🍀

```
#将文件内容拷贝到另外一个文件中

shutil.copyfileobj(open('sys.log','r'),open('new.log','w'))

#文件拷贝,如果目的文件不存在,则新建一个
shutil.copyfile('sys.log','sys_copy.log')

#仅拷贝权限..内容,组,用户均不变.目标文件必须实现存在
shutil.copymode('sys.log','sys_copy.log')

#仅拷贝状态的信息.目标文件必须实现存在
shutil.copystat('sys.log','sys_copy.log')

#拷贝文件和权限
shutil.copy('sys.log','sys_copy.log')

#递归拷贝目录.注意对目录父级目录要有可写权限，ignore的意思是排除.并且目标目录不能存在

shutil.copytree('module2','module2_copy',
                ignore=shutil.ignore_patterns("__init__.py"))


#拷贝软连接

shutil.copytree('module2','module2_copy',symlinks=True,
                ignore=shutil.ignore_patterns("__init__.py"))

#递归删除目录

shutil.rmtree('module2_copy')
```

* shutil模块创建压缩文件


创建压缩包并返回文件路径，例如：zip、tar

base_name： 压缩包的文件名，也可以是压缩包的路径。只是文件名时，则保存至当前目录，否则保存至指定路径，

如 data_bak         =>保存至当前路径

如：/tmp/data_bak   =>保存至/tmp/

format：	压缩包种类，“zip”, “tar”, “bztar”，“gztar”

root_dir：	要压缩的文件夹路径（默认当前目录）

owner：	    用户，默认当前用户

group：	    组，默认当前组

logger：	用于记录日志，通常是logging.Logger对象



* 打包module2目录为module2.tar.gz.放到当前文件

```
shutil.make_archive(base_name="module2",
                    format="gztar",
                    root_dir='module2')

```

* 解压shutile压缩包

shutil.unpack_archive('压缩包文件名',extract_dir='解压目标文件夹',format='压缩包格式') 

目标文件夹如果不存在,则自动创建

```
shutil.unpack_archive('module2.tar.gz',extract_dir='log',format='gztar')
```
