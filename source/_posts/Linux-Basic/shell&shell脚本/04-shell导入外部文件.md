---
title: shell脚本导入外部文件
date: 2018-06-12 22:59:58
tags:  Linux
categories: [Linux-Basic,shell&shell脚本 ]
comments: true
copyright: true
---

**shell脚本有时需要读取文件内容.下面介绍了从外部读取文件,引入循环语句处理该文件的几种常见方法**

**1.文本重定向**

```
定义一个文件内容:
[root@localhost ~]$cat users.cvs
tom
jesse
tony
```
shell脚本从该文件读取文件的每行内容,且创建相关账户

<!--more-->

```
[root@localhost ~]$cat users.sh
#!/bin/bash
#description: test the usage of input file into script.

input="users.cvs"      #定义一个变量,等于外部文件名

while read -r name    #while循环, read 读取文件内容,并且将内容赋值给name变量

do
echo "adding $name"      

useradd $name #创建用户

done < "$input"      #文件名内容通过重定向导入到while循环语句
```

执行脚本

```
[root@localhost ~]$bash users.sh
adding tom
adding jesse
adding tony
```

查看账户:

```
[root@localhost ~]$tail -3 /etc/passwd
tom:x:1000:1000::/home/tom:/bin/bash
jesse:x:1001:1001::/home/jesse:/bin/bash
tony:x:1002:1002::/home/tony:/bin/bash
```
> note: shell默认的分隔符是空格,或者换行符.这是通过变量IFS控制的.如果想要自定义其他分隔符,只需要定义IFS变量即可.例如:IFS=','

---

**2.通过管道符导入** 

还是刚才那个例子,文件内容和文件名不变.这次稍微改一下,将脚本的实现方式改成通过管道符导入,且将添加用户改成删除用户

```
[root@localhost ~]$cat users.sh
#!/bin/bash
#description: test the usage of input file into script


cat users.cvs | while read -r name    #while循环, read 读取文件内容,并且将内容赋值给name变量

do
echo "delete $name"      

userdel $name #删除用户

done 
```

执行结果

```
[root@localhost ~]$bash users.sh
delete tom
delete jesse
delete tony
[root@localhost ~]$tail -3 /etc/passwd
tss:x:59:59:Account used by the trousers package to sandbox the tcsd daemon:/dev/null:/sbin/nologin
postfix:x:89:89::/var/spool/postfix:/sbin/nologin
sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
```
---

**2.通过exec重定向导入** 

还是刚才那个例子,文件内容和文件名不变.这次稍微改一下,将用户重新添加回来

```
[root@localhost ~]$vim users.sh

#!/bin/bash
#description: test the usage of input file into script

exec 0< users.cvs

while read -r name    #while循环, read 读取文件内容,并且将内容赋值给name变量

do
echo "adding $name"      

useradd $name #删除用户

done
```

执行结果

```
[root@localhost ~]$bash users.sh
adding tom
adding jesse
adding tony

[root@localhost ~]$tail -3 /etc/passwd
tom:x:1000:1000::/home/tom:/bin/bash
jesse:x:1001:1001::/home/jesse:/bin/bash
tony:x:1002:1002::/home/tony:/bin/bash
```
> 0表示标准输入. exec 0 < 表示从文件里输入,而不是键盘输入



