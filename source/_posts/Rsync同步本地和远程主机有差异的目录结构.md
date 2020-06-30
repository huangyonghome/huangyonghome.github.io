### 需求

将BETA服务器的代码和配置文件数据迁移到IDC环境，IDC的BETA服务器已经存在一份数据，不能将本机所有的数据全部同步过去。一来耗时，占用带宽。二来会覆盖IDC BETA上正在使用的数据

所以，

1.需要判断如果对方（IDC BETA）如果不存在某个文件或者目录，则同步过去。

2.不能将目录上所有数据传递过去，因为代码目录包含很多不需要的旧版本releases的发布记录，全部同步过去耗费带宽和时间。


### 脚本内容

我直接在命令行操作的，没有写shell脚本。但是命令差不多

1.本地/data/apps/下的代码目录，如果对方服务器并不存在，则同步结构过去。

```
cd /data/apps

for dir in $(ls);do if ! `ssh work@172.16.20.1 "ls -d /data/apps/$dir" > /dev/null 2>&1`;then rsync -avz $dir  --include '*/' --exclude '*' work@172.16.20.1:/data/logs/;fi;done
```

以下是2个知识点:

```ssh work@iP COMMAND```可以在本地远程执行shell命令

以下命令只同步目录结构(包括所有子目录),但是不同步文件

```rsync -avz LOCAL_PATH --include '*/' --exclude '*' work@IP:REMOTE_PATH```



那么同样,同步nginx配置文件.这个时候因为是同步文件,所以使用scp命令.

```
 cd /etc/nginx/conf.d/
 
 for dir in $(ls);do if ! `ssh work@172.16.20.1 "ls  /data/conf/nginx/conf.d/$dir" > /dev/null 2>&1`;then scp $dir work@172.16.20.1:/data/conf/nginx/conf.d/;fi;done
 
 ```
 
 