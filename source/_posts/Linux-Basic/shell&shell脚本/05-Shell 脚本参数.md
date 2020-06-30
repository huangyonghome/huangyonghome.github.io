---
title: Shell的脚本参数
date: 2018-06-12 22:59:58
tags:  scripts
categories: [Linux-Basic,shell&shell脚本 ]
comments: true
copyright: true
---

## Shell的脚本参数

本章主要介绍2个命令

* **shift**
* **getopts**

---

#### 一.shift 

shift命令用于对参数的移动(左移)，通常用于在不知道传入参数个数的情况下依次遍历每个参数然后进行相应处理.

在使用shift时,默认情况他会将参数变量向左移动一个位置,所以变量$3会移动到$2,$2移动到$1,$1会被删除.($0是脚本名)

举个例子:

```
[root@localhost ~]$vim shift.sh

#!/bin/bash

count=1

while [ -n "$1" ];do
   echo "Parameter #$count = $1 "
   let count=$count+1
   shift
done
```
加入一些参数去执行这个脚本:

```
[root@localhost ~]$bash shift.sh jesse jerry jack tom tony
Parameter #1 = jesse 
Parameter #2 = jerry 
Parameter #3 = jack 
Parameter #4 = tom 
Parameter #5 = tony
```

> 可以看到每次echo一个参数后,shift就移掉最左边的一个参数,直到$1为空,退出循环

当然也可以一次移动多个位置.

稍微改动一下刚才的脚本

<!--more-->

```
[root@localhost ~]$vim shift.sh

#!/bin/bash

count=1

while [ -n "$1" ];do
   echo "Parameter #$count = $1 "
   let count=$count+1
   shift 2
done
```

执行:

```
[root@localhost ~]$bash shift.sh jesse jerry jack tom tony david
Parameter #1 = jesse 
Parameter #2 = jack 
Parameter #3 = tony 
```
---

### 二.getopts

getopts是getopt的升级版,增加了一些功能.getopts被shell程序用来分析位置参数，选项

getopts的格式:

getopts optstring variable

optstring是选项参数.选项字母被包含进optstring中.并将选项携带的参数添加进variable中.

getopts会用到2个环境变量.如果选项需要一个参数值,环境变量OPTARG就会保存这个值,OPTIND环境变量保存了参数列表中的getopts正在处理的参数位置.

一个简单的例子:

```
[root@localhost ~]$vim getopts.sh

#!/bin/bash

while getopts :ab:c opt
   do

       case "$opt" in
            a) echo "found the -a option";;
            b) echo "found the -b option,with value $OPTARG";;
            c) echo "found the -c option";;
            *) echo "unknown option: $opt";;
       esac
done
```

携带一些选项和参数来执行脚本.

```
[root@localhost ~]$bash getopts.sh -b "test" -a 
found the -b option,with value test
found the -a option

```
当然,也参数和选项之间也可以不用空格

```
[root@localhost ~]$bash getopts.sh -btest -a 
found the -b option,with value test
found the -a option
```
> getopts能从选项中识别出选项携带的值(test),除此之外还能将没有定义的选项输出成问号.

```
[root@localhost ~]$bash getopts.sh -d
unknown option: ?
```
>note 选项如果后面携带参数则需要在选项后添加冒号:  
换句话说,如果选项后面有冒号,则必须要携带一个参数值.

例如:

```
[root@localhost ~]$vim getopts1.sh

#!/bin/bash

while getopts b: opt
   do

       case "$opt" in
            b) echo "found the -b option,with value $OPTARG";;
            *) echo "unknown option: $opt";;
       esac
done
```

执行脚本,但是不携带参数:

```
[root@localhost ~]$bash getopts1.sh -b
getopts1.sh: option requires an argument -- b
unknown option: ?
```
携带一个参数:

```
[root@localhost ~]$bash getopts1.sh -b hello
found the -b option,with value hello
```
> 如果没有携带参数,则脚本会报错,且提示-b选项需要一个参数.

> note.以上脚本中选项没有以冒号开头,这个时候invalid option错误和miss option argument错误都会使varname被设成?，$OPTARG是出问题的option。 


再看一个例子.这次脚本的选项以:冒号开头

```
#!/bin/bash

while getopts :b: opt  #这次b选项以:开头
   do

       case "$opt" in
            b) echo "found the -b option,with value $OPTARG";;
            *) echo "unknown option: $opt";;
       esac
done
```

执行脚本,但是不携带参数:

```
[root@localhost ~]$bash getopts1.sh -b
unknown option: :
[root@localhost ~]$bash getopts1.sh -d
unknown option: ?

```

> 当optstring以”:”开头时，getopts会区分invalid option错误和miss option argument错误。  
invalid option时，varname会被设成?，$OPTARG是出问题的option；   
miss option argument时，varname会被设成:，$OPTARG是出问题的option。 

>上个例子中,-b选项没有携带参数.varname被设置成冒号:. -d选项没有定义,被设置成?


正常执行脚本,携带参数:

```
[root@localhost ~]$bash getopts1.sh -b hello
found the -b option,with value hello
```

---

#### getopts命令使用的完整范例

最后给出一个完整的脚本.次脚本完成了以下任务:

1.接收参数i,I,a    
2.-a选项的参数为interface接口,I选项参数为IP地址,-a选项不需要参数  
3.当用户使用-i选项,显示指定网卡接口的IP地址  
4.当用户使用-I选项,显示指定IP地址所属的网络接口  
5.当用户单独使用-a选项,显示所有网络接口和IP地址(环回接口lo除外)  

脚本内容

```

#!/bin/bash
#Description:
#Author: Jesse
#Version:0.1
#Datetime:2018-06-21
#Name:getinterface.sh
export PATH=/usr/lib64/qt-3.3/bin:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin


function SHOWIP(){
  if ! `ifconfig $1 &>/dev/null`;then
     return 11
  fi

  echo -n "$1's ip address is:"
   ifconfig $1 | grep "inet addr" |awk '{print $2}'|awk -F: '{print $2}'
}


function SHOWET() {
  if ! `ifconfig | grep "inet addr" |awk '{print $2}'|awk -F: '{print $2}'| grep $1 &>/dev/null`;then
     return 12

  fi

   echo -n "$1's interface is:"
   ifconfig | grep -B 1 "$1" | head -1 | awk '{print $1}'
}

function SHOWALL(){
  ETH=`ifconfig | grep ^[^[:space:]] | awk '{print $1}'|grep -v "lo" `

  for INTER in $ETH;do

      IP=`ifconfig $INTER| grep "inet addr" |awk '{print $2}'|awk -F: '{print $2}'`
      echo "$INTER:$IP"

  done

}

while getopts ":i:I:a" SWITCH;do
    case $SWITCH in

       i)
         SHOWIP $OPTARG
         [ $? -eq 11 ] && echo "you input the wrong Interface!"
         ;;
        I)
          echo $OPTARG | grep -E "^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$" &>/dev/null
          [ $? -ne 0 ] && echo "please input an valid IP address!" && exit 1

          SHOWET $OPTARG
          [ $? -eq 12 ] && echo "sorry,you input a wrong IP address "
          ;;
        a)
          SHOWALL
          ;;
        *)
          echo "Usage $0{-i interface|-I ip address|-a}"
          ;;
      esac

done

```

执行结果:

```
[root@oracle ~]# bash  getopts2.sh -i eth1
eth1's ip address is:172.16.1.17
[root@oracle ~]# bash  getopts2.sh -i eth0
you input the wrong Interface!
[root@oracle ~]# bash  getopts2.sh -I 172.16.1.17
172.16.1.17's interface is:eth1
[root@oracle ~]# bash  getopts2.sh -a
eth1:172.16.1.17
[root@oracle ~]# bash  getopts2.sh -i eth1 -a
eth1's ip address is:172.16.1.17
eth1:172.16.1.17
```
------

