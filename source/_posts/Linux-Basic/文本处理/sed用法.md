---
title: Sed介绍
date: 2018-06-12 22:59:58
tags:  Linux
categories: [Linux-Basic,文本处理 ]
comments: true
copyright: true
---



### sed 

* #### 概念:
Stream EDitor 流编辑器.sed以行为单位,纯文字符的行编辑器

sed:默认情况下不编辑原文件,而是提取需要处理的行到内存的模式空间去编辑,并且显示出来.

* #### sed 格式:  
sed [options]  ' AddressCommand' file      #sed [选项] '处理的地址块-命令' 文件

地址块和命令之间没有空格

* 地址块有下面几种方法定义:  

  

  <!--more-->

##### 1.起始行,结束行

比如:1,100---------表示第1,到第100行  
$表示最后一行  
+表示后续的行.比如1,+2 表示第1行,以后第2,3行

##### 2./正则表达式/ 

比如:/^root/ ---------表示以root为起始的行


##### 3./pattern1/,/pattern2/  

表示第一次被pattern1匹配到的行开始,到第一次被pattern2匹配到的行结束的中间所有行  

##### 4. /pattern/ 表示精准匹配pattern的所有行..


    注意:模式匹配需要双斜杠/ /把字符圈起来


###  sed用法

* ##### sed  [option]  ----------sed的选项,可以省略  
-n 使用安静模式,不输出模式空间中内容到屏幕  
-e 直接在命令行模式上进行动作编辑  
-f  直接将sed的动作写在一个文件内  
-r  支持扩展型正则表达式  
-i 直接修改读取的文件内容,不是由屏幕输出


* ##### Address---------------sed地址块  
[n1[,n2]] 

> n1,n2:不一定会存在,一般代表选择进行动作的行数.例如.如果是在10行和20行之间进行,那就是10,20 

* ##### Command命令  
a: 新增, a后面可以接字符串,而这些字符串会再新的一行出现  
c: 替换,c后面可以接字符串,这些字符串可以替换n1,n2之间的行  
d: 删除,因为是删除,所以d后面通常不接任何参数  
i:  插入,i的后面可以接字符串,而这些字符串会再新的一行出现  
p:打印,也就是讲某个选择的数据打印出来,通常会与参数-n 一起执行  
s: 替换,可以直接进行替换的工作,通常这个s的动作可以搭配正则表达式!  
!: 取反


> 1.默认情况下,只替换每行中第一次匹配到的字符串.加上修饰符g,则表示如果行中还有其他也能匹配则继续替换.例如:
>
> 1,20s/old/new/g
>
> 2.默认情况下可以使用/作为分隔符,当然也可以使用其他符号作为分隔符.比如:s#old#new#.或者s@old@new@
>
>3.s/pattern/string/ 前面可以用正则表达式查找模式.但是替换后的字符串,则不能使用正则表达式


#### sed用法例子:

##### 基础用法:  

* ##### 将2-5行删除

注意,sed的动作,需要2个单引号括住:

    [root@localhost ~]$nl /tmp/passwd | sed '2,5d'
     1	root:x:0:0:root:/root:/bin/bash
     6	sync:x:5:0:sync:/sbin:/bin/sync
     7	shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
     8	halt:x:7:0:halt:/sbin:/sbin/halt

* ##### 删除3,到最后一行.$代表最后一行

```
    [root@localhost ~]$nl /tmp/passwd | sed '3,$d'
    1	root:x:0:0:root:/root:/bin/bash
    2	bin:x:1:1:bin:/bin:/sbin/nologin
```
* ##### 在第二行后面新增一行
```
    [root@localhost ~]$nl /tmp/passwd | sed '2a this is a new line'
    1	root:x:0:0:root:/root:/bin/bash
    2	bin:x:1:1:bin:/bin:/sbin/nologin
    this is a new line
    3	daemon:x:2:2:daemon:/sbin:/sbin/nologin  
```
* ##### 在第二行前新增,并且新增2行  
``` 
[root@localhost ~]$nl /tmp/passwd | sed '2i this is a line before 2 \
> continue a line '
     1	root:x:0:0:root:/root:/bin/bash
this is a line before 2 
continue a line 
     2	bin:x:1:1:bin:/bin:/sbin/nologin
```
* ##### 将2-5行内容替换成为"no 2-5 number"

```
[root@localhost ~]$nl /tmp/passwd | sed '2,5c no 2-5 number'
     1	root:x:0:0:root:/root:/bin/bash
no 2-5 number
     6	sync:x:5:0:sync:/sbin:/bin/sync
```

* ##### 用 sed -n的参数,只取出5-7行数据

```
[root@localhost ~]$nl /tmp/passwd | sed -n '2,5p'
     2	bin:x:1:1:bin:/bin:/sbin/nologin
     3	daemon:x:2:2:daemon:/sbin:/sbin/nologin
     4	adm:x:3:4:adm:/var/adm:/sbin/nologin
     5	lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
[root@localhost ~]$
```

#### sed直接修改文件

-i 参数.. 代表直接修改文件,而不输出到屏幕上  

* 例如:将下列文件中的小数点换成逗号
```
[root@www tmp]$cat 2.txt
haha....sed
[root@www tmp]$sed -i 's/\./,/g' 2.txt
[root@www tmp]$cat 2.txt
haha,,,,sed
```


#### 替换用法

格式: sed 's/要被替换的字符串/新的字符串/g'

抓取IP地址:
```
[root@www ~]$ifconfig eth0 | grep "inet addr"
          inet addr:172.16.1.120  Bcast:172.16.1.255  Mask:255.255.255.0
```

##### sed删除.^.*addr: 代表任意字符开头的,addr:结尾的字符串. //代表空  
```
[root@www ~]$ifconfig eth0 | grep "inet addr" | sed 's/^.*addr://g'
172.16.1.120  Bcast:172.16.1.255  Mask:255.255.255.0
```
##### 删除Bcast的后半部分

```
[root@www ~]$ifconfig eth0 | grep "inet addr" | sed 's/Bcast.*$//g'
          inet addr:172.16.1.120  
```
##### 只保留IP,其余部分删除
```
[root@www ~]$ifconfig eth0 | grep "inet addr" | sed 's/Bcast.*$//g' | sed 's/^.*addr://g'
172.16.1.120 
```
##### 删除以#开头的,以及空白行  
```
[root@www ~]$cat /etc/man.config | grep 'MAN' | sed 's/^#.*$//g' | sed '/^$/d'
MANPATH	/usr/man
MANPATH	/usr/share/man
MANPATH	/usr/local/man
MANPATH	/usr/local/share/man
MANPATH	/usr/X11R6/man
MANPATH_MAP	/bin			/usr/share/man
MANPATH_MAP	/sbin			/usr/share/man
MANPATH_MAP	/usr/bin		/usr/share/man
MANPATH_MAP	/usr/sbin		/usr/share/man
MANPATH_MAP	/usr/local/bin		/usr/local/share/man
MANPATH_MAP	/usr/local/sbin		/usr/local/share/man
MANPATH_MAP	/usr/X11R6/bin		/usr/X11R6/man
MANPATH_MAP	/usr/bin/X11		/usr/X11R6/man
MANPATH_MAP	/usr/bin/mh		/usr/share/man
MANSECT		1:1p:8:2:3:3p:4:5:6:7:9:0p:n:l:p:o:1x:2x:3x:4x:5x:6x:7x:8x
```
#### 马哥的Linux视频sed练习

以下是原文

```
hello world
  hello jesse
hello jesse's world
id:3:initdefault:  do you know id?


#i love you ,do you know
#i love you very much
#  This is a test,
#   there are some spaces follow #
      #there are some spaces in the front of this line
      #and there is a string # follows the space


```

* ##### 练习1:删除行首中的空白符,也就是1,2,11,12行的行首空白符删掉.

```
[root@www tmp]$sed 's/^[[:space:]]*//g' test.txt
hello world
hello jesse
hello jesse's world
id:3:initdefault:  do you know id?


#i love you ,do you know
#i love you very much
#  This is a test,
#   there are some spaces follow #
#there are some spaces in the front of this line
#and there is a string # follows the space
[root@www tmp]$
```
* ##### 练习2.替换"id:3:initdefault:" 中的数字3为5

```
[root@www tmp]$sed 's/id:3:initdefault:/id:5:initdefault:/' test.txt
    hello world
     hello jesse
   hello jesse's world
   id:5:initdefault:  do you know id?
```

* #####  练习3.删除空白行
解释:查找模式/^$/ ,d参数表示删除.注意,要查找的内容必须用//括起来

```
[root@localhost ~]$sed '/^$/d' /tmp/test.txt
    hello world
     hello jesse
   hello jesse's world
   id:3:initdefault:  do you know id?
   #i love you ,do you know
   #i love you very much
   #  This is a test,
  #   there are some spaces follow #
      #there are some spaces in the front of this line
      #and there is a string # follows the space
```

* #####  删除文件中开头的#号  

```
[root@localhost ~]$sed 's/^#//g' /tmp/test.txt
hello world
  hello jesse
hello jesse's world
id:3:initdefault:  do you know id?


i love you ,do you know
i love you very much
```

* #####  练习5.删除开头的#号及后面的空白字符.也就是删除9,10两行开头的#和空格

> 解释:这里用两个空格字符,是表示#空格然后再*重复前面的空格0次或者多次.  
如果是sed 's/^#[[:space:]]*//' sed.txt 则表示空格字符可以是0次,也可以是无穷次,会把上面两行#i love you一起删除


```
[root@localhost ~]$sed 's/^#[[:space:]][[:space:]]*//g' /tmp/test.txt
hello world
  hello jesse
hello jesse's world
id:3:initdefault:  do you know id?


#i love you ,do you know
#i love you very much
This is a test,
there are some spaces follow #
      #there are some spaces in the front of this line
      #and there is a string # follows the space
```

或者可以用扩展正则

```
[root@localhost ~]$sed -r 's/^#[[:space:]]+//g' /tmp/test.txt
hello world
  hello jesse
hello jesse's world
id:3:initdefault:  do you know id?


#i love you ,do you know
#i love you very much
This is a test,
there are some spaces follow #
      #there are some spaces in the front of this line
      #and there is a string # follows the space
```

> NOTE:扩展正则需要加上-r参数

* #####  练习6.删除空白字符后面跟#字符,也就是最后两行
和刚刚相反即可

```
[root@localhost ~]$sed -r 's/^[[:space:]]+#//g' /tmp/test.txt
hello world
  hello jesse
hello jesse's world
id:3:initdefault:  do you know id?


#i love you ,do you know
#i love you very much
#  This is a test,
#   there are some spaces follow #
there are some spaces in the front of this line
and there is a string # follows the space
```

### sed位置替换

##### sed在替换的时候支持位置参数.比如看下面2个例子:

#####  1.将下列所有以_all结尾的单词前面加个>破折号.

```
cat test
fsdafdsa_all
weewwe_all
haha_all
afsdasdf
2232333333333
```

```
[root@localhost tmp]$sed -r 's@(.*)_all$@\1>_all@g' pos.txt
cat test
fsdafdsa>_all
weewwe>_all
haha>_all
afsdasdf
2232333333333
```

##### 解析 sed -r 's@(.*)_all@>\1_all@g':

-r 表示扩展正则,这里虽然没有用到扩展正则表达式,但是位置变量替换本身就是扩展正则  
(.*) 表示匹配任意字符  
\1 表示代替前面的第一个(.*)正则表达式..也就是说把前面这个.*匹配到的所有内容放在\1这个位置上.  
所以这个语句表示:匹配任意以_all结尾的字符串,然后保留\1分组匹配到的(.\*)内容不变,只是在\_all前面加个>破折号  
所以前面.*匹配到的fsdafdsa,weewwe,haha会被放置在>破折号后面,_all之前.

##### 2.将一个文件中的taomee、******、peoplenet中的*内容进行替换成network（*的内容不同）

```
[root@localhost tmp]$echo "taomee,stage,peonetwork" > 1.txt
[root@localhost tmp]$cat 1.txt
taomee,stage,peonetwork
[root@localhost tmp]$sed -r 's#(.*),.{5},(.*)#\1,network,\2#' 1.txt
taomee,network,peonetwork
```

##### 解析: sed -ri 's#(.*),.{5},(.*)#\1,network,\2#g'  
.{5} .代表一个任意字符,{5}表示重复前一个匹配5次,这里就是匹配5个任意字符  
\1 表示第一个正则匹配的内容放入的位置.这里表示第一个.*  
\2 表示第二个正则匹配的内容放入的位置,这里表示第二个.*  
network:表示.{5}匹配的内容,替换成network  
所以这里就是表示下列各式的内容:任意字符 5个任意字符 任意字符  然后把中间5个任意字符替换成network

##### 3.将下列 database_host: 10.0.0.10的IP改成10.0.0.2:


```
work@docker$ cat parameters.yml
# This file is auto-generated during the composer install
parameters:
    database_host: 10.0.0.10
```

```
sed -re '/database_host:/s#(:\s*).+$#\110.0.0.2#' parameters.yml
```

##### 解析:/s#(:\s*).+$#\110.0.0.3# : 
\1表示匹配第一个分组  
(:\s*)所匹配到的内容保留下来.也就是保留:和10这中间的空格部分  
10.0.0.2代替.+匹配到的内容,也就是用10.0.0.2替代匹配到的10.0.0.10
