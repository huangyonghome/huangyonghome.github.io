---
title: bash变量基础知识  
date: 2018-06-14 00:44:48  
tags: scripts
categories: [Linux-Basic,shell&shell脚本 ]  
comments: true  
copyright: true  
---

## bash变量
---

本章内容主要包括

1.简介  
2.变量打印  
3.变量赋值   
4.变量声明  
5.变量内容的删除,替代  
6.变量运算表达式


### 1.简介

---

变量是shell脚本中必不可少的组成部分，在脚本中使用变量不需要提前声明。在bash中每一个变量都是字符串，所以在变量赋值时候不管有没有使用引号都是以字符串的形式存储 

---

<!--more-->

### 2.变量打印

echo $变量名  
或者:  
echo ${变量名}
```
[root@localhost ~]$echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
```
---

### 3.变量赋值

- 语法:  
  变量名=变量内容 

设置一个变量myname. 值等于jesse:

```
[root@localhost ~]$myname=jesse
[root@localhost ~]$echo $myname
jesse
```

#### 变量赋值规则和方法

1.等号两边不能直接接空格符  
2.变量名只能是英文字母与数字,但是开头字符不能是数字  
3.变量内容若有空格符可使用双引号或者单引号将变量内容结合起来.

>Note: 双引号内的特殊字符如$,可以保持原本的特性.单引号内的特殊字符则仅为一般字符.

4.可以用转义符"\"将特殊符号(如enter,$,空格等)变成一般字符  
5.在一串命令中,还需要使用其他命令,可以使用如下格式:  
> - 反单引号`命令`-------------------注意这个是键盘上数字1左边的引号.不是单引号
> - $(命令)

示例:

- 变量左右两边不能有空格,或者变量以数字,特殊符号开头

```
[root@localhost ~]$my = jesse
-bash: my: command not found
[root@localhost ~]$-my=jesse
-bash: -my=jesse: command not found
[root@localhost ~]$123my=jesse
-bash: 123my=jesse: command not found
```
- 变量值如果嵌套另外一个变量的值.需要用双引号,如果用单引号则变量符$只是单纯字符,如:

```
[root@localhost ~]$echo $LANG
en_US.UTF-8
[root@localhost ~]$my="lang is $LANG" ; echo $my
lang is en_US.UTF-8
[root@localhost ~]$my='lang is $LANG' ; echo $my
lang is $LANG
```
- 也可以用\转移符,来转义特殊字符变成一般字符:  

```
[root@localhost ~]$my="lang is \$LANG" ; echo $my
lang is $LANG
```
> 如果变量值的引号需要成双配对,如果是包含一个引号,或者空格,必须要转义.

```
[root@localhost ~]$name=jesse\'s;echo $name
jesse's
[root@localhost ~]$name=jesse\'s\ huang;echo $name
jesse's huang

```

- 如果变量的值为某一个命令执行的结果,则有下列两种方式实现:

```
1.反引号:  
[root@localhost ~]$myname=`hostname`;echo $myname
localhost.localdomain

2.$(command):
[root@localhost ~]$myname=$(cat /etc/redhat-release);echo $myname
CentOS Linux release 7.2.1511 (Core)
```

6.若该变量增加变量内容时,可以用$变量名称或${变量}累加,例如

```
[root@localhost ~]$myname=${myname}huang;echo $myname
jessehuang
[root@localhost ~]$myname=$myname+408;echo $myname
jessehuang+408
```
> 这里可以看到,如果是$myname和huang两个字符串无缝拼接,需要将变量名用{}大括号括起来,如果只是写成$jessehuang,na那么会将$mynamehuang作为一个变量名处理.

7.如果该变量需要在其他进程执行,需要用export来使变量变成环境变量

export 变量名

> note: export 只需要跟变量名.不需要$变量名

8.取消变量的方法为使用unset 变量名称  

unset 变量名

9.变量还可以通过键盘读取.这在交互式脚本中经常使用

```
[root@localhost ~]$read myname
jesse
[root@localhost ~]$echo $myname
jesse

还可以使用-p参数,输入提示符
[root@localhost ~]$read -p "Please input your name: " myname
Please input your name: jesse
[root@localhost ~]$echo $myname
jesse
```
#### 变量单引号和双引号的区别



单引号是里面是啥结果就是啥，赋值后就是你等号后面去掉最开始的单引号和结尾的单引号

双引号是里面有变量就展开， 赋值后就是你等号后面去掉最开始的双引号和结尾的双引号 

例如下面的例子:

1.双引号内的变量会展开,赋值后去掉开始的双引号和结尾的双引号.

```
[root@yunwei-test ~]$a="a b c d"
[root@yunwei-test ~]$b="var a is $a"
[root@yunwei-test ~]$echo $b
var a is a b c d
[root@yunwei-test ~]$
```

2.单引号内的变量不会展开,复制后去掉开始的单引号和结尾的单引号..下面的例子中变量c没有展开

```
[root@yunwei-test ~]$c='a b c d'
[root@yunwei-test ~]$d='var c is $c'
[root@yunwei-test ~]$echo $d
var c is $c
```

3.赋值后如果想要保留双引号或者单引号.可以在变量前加个单/双引号

```
[root@yunwei-test ~]$e='"a b c d"'
[root@yunwei-test ~]$f="var e is $e"
[root@yunwei-test ~]$echo $f
var e is "a b c d"

[root@yunwei-test ~]$e="'a b c d'"
[root@yunwei-test ~]$f="var e is $e"
[root@yunwei-test ~]$echo $f
var e is 'a b c d'

```

---

### 4.变量声明

declare 或者typeset

declare用法:

> declare [-aixr] 变量名

-a 将变量定义成为数组(array)类型  
-i  将变量定义为整数(integer)类型  
-x 用法和export一样 就是将后面的变量变成环境变量  
-r  将变量设置成为readonly,不可更改,也不能重设  

- 示例.让sum变量等于数字累加的结果:

```
[root@localhost ~]$sum=100+200
[root@localhost ~]$echo $sum
100+200
[root@localhost ~]$declare -i sum=100+200;echo $sum
300
```

> 字符串+没有特别声明,不会进行操作符的操作.只是普通字符串

declare 的x参数还能将变量定义为环境变量.  
-x 表示定义为环境变量
+x 表示取消为环境变量

```
[root@localhost ~]$declare -x sum
[root@localhost ~]$export | grep sum
declare -ix sum="200"
[root@localhost ~]$declare +x sum
[root@localhost ~]$export | grep sum
[root@localhost ~]$
```
---

### 5.变量内容的删除,替代

#### 5.1.1. 变量删除


用法1:

```
${变量名#}-------------#代表从变量内容的最前面开始向右删除,,且仅删除最短的那个
```

- 删除PATH变量的第一个路径

```
[root@localhost ~]$echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
[root@localhost ~]$mypath=${PATH#/usr/local/sbin:};echo $mypath
/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
```
- 下面的例子虽然表示删除所有的/*:的字符串,但是只删除了第一个字符串

```
root@localhost ~]$mypath=${PATH#/*:};echo $mypath
/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
```

用法2:  

```
${变量名##}-------------两个#代表从变量内容的最前面开始向右删除,,且删除所有匹配字符
```

```
[root@localhost ~]$mypath=${PATH##/*:};echo $mypath
/root/bin
```

用法3:

```
${变量名%}----------------%代表从变量内容的最后面开始向前删除,且删除最短字符,
```


- 在下个例子中只删除了分隔符:的最右侧/usr/bin

```
[root@localhost ~]$echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
[root@localhost ~]$mypath=${PATH%:*};echo $mypath
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin.

```

用法4:

```
${变量名%%}----------------%%代表从变量内容的最后面开始向前删除,且删除最长的符合字符串,
```

- 在下面的例子中,从最右侧一直删除所有分隔符:的右侧内容

```
[root@localhost ~]$mypath=${PATH%%:*};echo $mypath
/usr/local/sbin
```

- 以下例子删除了所有,无论是从左到右还是从右到左删除  

```
[root@localhost ~]$mypath=${PATH%%*};echo $mypath

[root@localhost ~]$mypath=${PATH##*};echo $mypath

[root@localhost ~]$
```
---

#### 5.1.2. 变量替换

用法1

> ${变量名/旧变量内容/新变量内容}-----------单/表示只替换1个

- 将PATH变量中的sbin替换成SBIN.下列中只替换了一个sbin

```
[root@localhost ~]$mypath=${PATH/sbin/SBIN};echo $mypath
/usr/local/SBIN:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
```
用法2

>${变量名//旧变量内容//新变量内容}-----------双//表示替换所有匹配内容

```
[root@localhost ~]$mypath=${PATH//sbin/SBIN};echo $mypath
/usr/local/SBIN:/usr/local/bin:/usr/SBIN:/usr/bin:/root/bin
```
---

### 6.变量的运算表达式


- let命令

```
[root@localhost ~]$A=3
[root@localhost ~]$B=4
[root@localhost ~]$let C=$A+$B
[root@localhost ~]$echo $C
7
```

- 中括号

```
[root@localhost ~]$C=$[$A+$B];echo $C
7
```
-  双括号

```
[root@localhost ~]$C=$(($A+$B));echo $C
7
```

- 还有上面提到的delcare变量声明语句

```
[root@localhost ~]$C=$A+$B;echo $C
3+4
[root@localhost ~]$declare -i C=$A+$B;echo $C
7
```

