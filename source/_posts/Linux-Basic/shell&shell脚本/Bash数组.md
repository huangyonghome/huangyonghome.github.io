---
title: Bash数组  
date: 2018-06-12 22:59:58  
tags:  scripts
categories: [Linux-Basic,shell&shell脚本 ]  
comments: true  
copyright: true  
---

## Bash数组

### Bash中的数组介绍

数组中可以存放多个值。Bash Shell 只支持一维数组（不支持多维数组）。

与大部分编程语言类似，数组元素的下标由0开始。

---

<!--more-->

### 数组的创建

Shell 数组用括号来表示，元素用"空格"符号分割开，语法格式如下：

```
array_name=(value1 ... valuen)
```

实例:

创建一个数组变量的方法:

```
[root@localhost ~]$A=(a b c d)
```
> 注意,元素之间没有逗号


或者可以用下标来定义数组

```
[root@localhost ~]$B[0]=1
[root@localhost ~]$B[1]=2
[root@localhost ~]$B[2]=3
```
> Tips 当然用这种方式也可以修改数组的某个元素值.比如将第1个元素的值由1改成0:  
[root@localhost ~]$B[0]=0

---

### 数组读取

1.读取单个元素.

语法:

```
${array_name[index]}
```
例如.读取A数组的第二个元素

```
[root@localhost ~]$echo "${A[2]}"
c
```

2. 读取所有元素.

使用@ 或 * 可以获取数组中的所有元素，例如：

```
[root@localhost ~]$echo ${A[*]}
a b c d
[root@localhost ~]$echo ${A[@]}
a b c d
```

3.读取数组的长度

获取数组长度的方法与获取字符串长度的方法相同，语法:

```
[root@localhost ~]$echo ${#A[@]}
4
[root@localhost ~]$echo ${#A[*]}
4
```

### 数组删除

1. 删除整个数组

语法: unset 数组名

例如:

```
[root@localhost ~]$unset A
[root@localhost ~]$echo ${A[*]}

[root@localhost ~]$
```
2.删除指定的下标元素

语法: unset 数组名[index]

例如.删除最后一个元素

```
[root@localhost ~]$A=(a b c d)
[root@localhost ~]$unset A[3]
[root@localhost ~]$echo ${A[*]}
a b c
```

### 数组分片

像字符串截取或者Python的列表分片一样.数组也支持分片.

语法:

```
{arry[* or @]:起始分片位置:长度}
```

例如.将数组A保留前面3个元素

```
[root@localhost ~]$echo ${A[*]:0:3}
a b c
```

长度可以省略.例如下面表示从1个位置开始分片

```
[root@localhost ~]$echo ${A[*]:1}
b c d e
```

### 数组替换

元素值的替换

语法:

```
{arry[* or @]:查找字符:替换字符}
```

例如: 将第数组A的c替换成1

```
[root@localhost ~]$echo ${A[*]}
a b c d e
[root@localhost ~]$echo ${A[*]/c/1}
a b 1 d e
```

如果是一个数组里有重复的多个相同元素,则被全部替换.例如:

```
[root@localhost ~]$A=(a b c d e c c)
[root@localhost ~]$echo ${A[*]/c/1}
a b 1 d e 1 1
```

### 数组追加元素

例如:  
ary2=(1 2 3 4 5 6) .要追加7 8 9 进去:

```
[root@localhost ~]$vim arry1.sh

#!/bin/bash

ary2=(1 2 3 4 5 6)

index=6 #之前的ary本身已经有6个值,所以索引从6开始

for i in 7 8 9;do  #如果for循环的项不多,可以直接写在In 后面
   ary2[$index]=$i
   index=$index+1
done

echo "Now ary2 is: ${ary2[*]}"
```

执行脚本:

```
[root@localhost ~]$bash arry1.sh 
Now ary2 is: 1 2 3 4 5 6 7 8 9
```

## 第二部分.数组判断

判断一个变量是否存在数组内.

语法:

```
[[ ${ary2[@]/$a/} == ${ary2[@]} ]] 
```
原理是借鉴了数组分片的方法:将数组中去掉某个元素,如果和之前的数组不相等.则肯定此元素存在数组中.  
如果去掉某个元素,仍然和之前的数组相等.那么此元素就不属于数组中   

例如:

判断元素3是否在数组ary2中:

```
[root@localhost ~]$ary2=(1 2 3 4 5 6)

[root@localhost ~]$[[ ${ary2[*]/3/} != ${ary2[*]} ]] && echo "3 is in ary2" || echo "3 is not in ary2"
3 is in ary2
```

判断元素10是否在数组ary2中:

```
[root@localhost ~]$[[ ${ary2[*]/10/} != ${ary2[*]} ]] && echo "10 is in ary2" || echo "10 is not in ary2"
10 is not in ary2
```


### 举一个数组应用的例子:

读取下列IP文件,并赋值给一个数组:

```
[root@localhost ~]$cat ip.txt 
10.0.0.1
10.0.0.2
10.0.0.3
10.0.0.4
10.0.0.5
```

脚本:

```
#!/bin/bash

index=0

while read line;do
   arry[$index]=$line
   index+=1
done< <(cat ip.txt)

echo " arry is ${arry[*]}"
echo " arry has ${#arry[*]} elements"
```

执行脚本:

```
[root@localhost ~]$bash ary.sh
 arry is 10.0.0.1 10.0.0.2 10.0.0.3 10.0.0.4 10.0.0.5 
 arry has 6 elements
```










