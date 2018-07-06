---
title: 文本工具介绍
date: 2018-06-12 22:59:58
tags:  Linux
categories: [Linux-Basic,文本处理 ]
comments: true
copyright: true
---

### 常见文本工具使用

---

#### 管道命令介绍
---

1.管道|后面必须紧接一个命令.  
2.管道命令仅仅会处理标准输出的内容,对于错误输出的不能处理  
3.管道命令必须能够接收前一个命令输入的信息(standard input),比如less,more,head,tail.而ls,cp,mv则不行  
4.管道命令必须能够处理前一个命令执行后的结果,

<!--more-->

***


#### 1. cut
---

用法:

cut -d '分隔字符' -f fields  
cut -c 字符范围


-d 后面接分隔字符,与-f一起使用 -----------注意,分隔符要用单引号  
-f 依据-d的分隔字符,将一段信息切割成为数段,用-f 表示取出第几段  
-c 以字符的单位去除固定字符区间  

示例:

* 取出前3行的用户名和UID

```
[root@localhost ~]$cat /etc/passwd | cut -d ':' -f 1,3 | head -3
root:0
bin:1
daemon:2
```
- 也可以对一行文本处理,例如,取出PATH变量的第3和第5个值

```
[root@localhost ~]$echo $PATH | cut -d ':' -f 3,5
/usr/sbin:/root/bin
```
- 取出last的第一行内容.此时是空格为分隔符

```
[root@localhost ~]$last | cut -d ' ' -f 1
root
root
root
root
reboot
root
```
---

#### 2.排序命令
---

##### 2.1 sort

Sort 用法


sort [-fbMnrtuk] 

-f  忽略大小写  
-b 忽略最前面的空格符  
-n 使用数字进行排序,默认是文字  
-r 反向排序  
-u 相同的数据,仅出现一行  
-t 分隔符,默认是用tab键分隔  
-k 以那个区间进行排序  

示例:

- 如果没有-n参数,则默认按首字母排序

```
[root@localhost ~]$head -5 /etc/passwd | sort
adm:x:3:4:adm:/var/adm:/sbin/nologin
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
root:x:0:0:root:/root:/bin/bash
```
- 以第三列的UID来排序

```
[root@localhost ~]$head -5 /etc/passwd | sort -nt ':' -k 3
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
```
- 只抓取第一和第三列,且以数字的倒序来排序

```
[root@localhost ~]$head -5 /etc/passwd | cut -d ':' -f 1,3 | sort -t ':' -nr  -k 2
lp:4
adm:3
daemon:2
bin:1
root:0
```
> 注意经过cut处理后,原本处于第三列的UID值已经变成第二列了
---

#### - uniq
---

用法: uniq [-ic]

-i 忽略大小写字符的不同  
-c 进行计数

> NOTE:uniq一般和sort一起使用,而且sort要出现在Uniq出现.先排序再计数,切记顺序不要搞反.

示例:

- 使用last将账户列出,排序后,同一账号进出现一次

```
[root@localhost ~]$last | cut -d ' ' -f 1 | sort |uniq

reboot
root
wtmp
```
- 在此基础上计算各账号登录了多少次

```
[root@localhost ~]$last | cut -d ' ' -f 1 | sort |uniq -c
      1 
      5 reboot
      8 root
      1 wtmp
```
---

#### - wc
---

用法:

wc [-lwm]  
-l 仅列出行  
-w 仅列出多少字  
-m 列出多少字符

示例:

- wc如果没带任何参数,默认会列出三个数字.  
分别表示行数,字数,字符数.例如:

```
[root@localhost ~]$cat /etc/passwd |wc
     21      43    1040
```

- 一般wc用户计算行数.例如,计算当前目录下有多少个文本

```
[root@localhost ~]$ls | wc -l
3
```



