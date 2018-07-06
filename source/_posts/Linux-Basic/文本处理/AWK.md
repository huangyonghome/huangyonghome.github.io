---
title: AWK介绍
date: 2018-06-12 22:59:58
tags:  Linux
categories: [Linux-Basic,文本处理 ]
comments: true
copyright: true
---

### AWK

#### 简介

1.概念  
2.基础格式  
3.输出,格式化输出   
4.awk操作符      
5.awk变量    
6.awk模式    
7.控制语句  
8.数组  
9.经典例子  


#### 概念

awk是一个强大的文本分析工具，相对于grep的查找，sed的编辑，awk在其对数据分析并生成报告时，显得尤为强大。简单来说awk就是把文件逐行的读入，以空格为默认分隔符将每行切片，切开的部分再进行各种分析处理。

awk有3个不同版本: awk、nawk和gawk，未作特别说明，一般指gawk，gawk 是 AWK 的 GNU 版本。

<!--more-->

awk其名称得自于它的创始人 Alfred Aho 、Peter Weinberger 和 Brian Kernighan 姓氏的首个字母。实际上 AWK 的确拥有自己的语言： AWK 程序设计语言 ， 三位创建者已将它正式定义为“样式扫描和处理语言”。它允许您创建简短的程序，这些程序读取输入文件、为数据排序、处理数据、对输入执行计算以及生成报 表，还有无数其他的功能。

---

#### 基础格式

###### awk '{pattern + action}' {filenames}   
###### awk [options] 'BEGIN{ action;… } pattern{ action;… } END{ action;… }' file ...

---

#### 输出,格式化输出

 ##### 1. print

print的使用格式：  

> print item1, item2, ...  

要点：  

1、各项目之间使用逗号隔开，而输出时则以空白字符分隔；

2、输出的item可以为字符串或数值、当前记录的字段(如$1)、变量或awk的表达式；数值会先转换为字符串，而后再输出；

3、print命令后面的item可以省略，此时其功能相当于print $0, 因此，如果想输出空白行，则需要使用print ""；

例如:

>awk -F: '{ print $1, $3 }' /etc/passwd

##### 2. printf 格式化输出

- printf命令的使用格式：

>printf format, item1, item2, ...

要点：

1、其与print命令的最大不同是，printf需要指定format；  
2、format用于指定后面的每个item的输出格式；  
3、printf语句不会自动打印换行符；\n  

- format格式的指示符都以%开头，后跟一个字符；如下：

%c: 显示字符的ASCII码；  
%d, %i：十进制整数；  
%e, %E：科学计数法显示数值；  
%f: 显示浮点数；  
%g, %G: 以科学计数法的格式或浮点数的格式显示数值;  
%s: 显示字符串；  
%u: 无符号整数；  
%%: 显示%自身；  

- 修饰符：  

N: 显示宽度；  
-: 左对齐；  
+：显示数值符号； 


例子：

```
[root@www tmp]$awk -F: '{printf "%-15s %i\n",$1,$3}' /etc/passwd
root            0
bin             1
daemon          2
adm             3
lp              4
sync            5
shutdown        6
halt            7
mail            8
uucp            10

%-15s:左对齐15个宽度,并且为%s字符串  
%i: 第二列为整数
\n: 不自动换行
```

例如:

```
[root@www tmp]$awk -F: '{printf "Username: %-15s UID:%d\n",$1,$3}' /etc/passwd
Username: root            UID:0
Username: bin             UID:1
Username: daemon          UID:2
Username: adm             UID:3
Username: lp              UID:4
Username: sync            UID:5
Username: shutdown        UID:6
```
---

#### awk操作符

- 算术操作符：x+y, x-y, x*y, x/y, x^y, x%y  
- 赋值操作符：=, +=, -=, *=, /=, %=, ^=，++, --  
- 比较操作符：==, !=, >, >=, <, <=  
- 模式匹配符：~ ：左边是否和右边匹配包含；!~ ：是否不匹配  
- 逻辑操作符：与:&& ；或:|| ；非:!  
- 条件表达式（三目表达式）：selector ? if-true-expression : if-false-expression

例如:

```
[root@www tmp]$awk -F":" '$3>100 {print $1,$3}' /etc/passwd
abrt 173
nfsnobody 65534
saslauth 499
jesse 500
tomcat 502
ansible 503
work 1500
[root@www tmp]$awk -F":" '($3>100) {print $1,$3}' /etc/passwd
abrt 173
nfsnobody 65534
saslauth 499
jesse 500
tomcat 502
ansible 503
work 1500
```
#### awk常用变量

- FS: field separator，读取文件本时，所使用字段分隔符；  
- RS: Record separator，输入文本信息所使用的换行符；  
- NR: The number of input  records，awk命令所处理的记录数；如果有多个文件，这个数目会把处理的多个文件中行统一计数；  
- FNR: 与NR不同的是，FNR用于记录正处理的行是当前这一文件中被总共处理的行数；  
- NF：Number of Field，当前记录的field个数；也就是有多少列  
- FILENAME: awk命令所处理的文件的名称；  

###### 用户自定义变量

\-v参数可以指定变量,比如 awk -v a=10 '{print $a}'

例如:

```
[root@www tmp]$awk -F":" -v a=3 ' NR==1 {print $1,$a}' /etc/passwd
root 0
```
---

#### awk模式

也就是pattern { action }

##### 常见的模式类型： 

1.Regexp: 正则表达式，格式为/regular expression/  
2.expresssion： 表达式，其值非0或为非空字符时满足条件，如：$1 ~ /foo/ 或 $1 == "magedu"，用运算符~(匹配)和!~(不匹配)。  
3.Ranges： 指定的匹配范围，格式为pat1,pat2  
4.BEGIN/END：特殊模式，仅在awk命令执行前运行一次或结束前运行一次.

> BEGIN：让用户指定在第一条输入记录被处理之前所发生的动作，通常可在这里设置全局变量。  
>END：让用户在最后一条输入记录被读取之后发生的动作。

5.Empty(空模式)：匹配任意输入行；


- 例子1.表达式作为模式,下例子匹配$3大于1000且小于2000

```
[root@www tmp]$awk -F: '$3>1000 && $3<2000 {print $1,$3}' /etc/passwd
work 1500
```

- 例子2.正则匹配

```
[root@www tmp]$awk -F: '/^[a-g]/ {print $1,$3}' /etc/passwd
bin 1
daemon 2
adm 3
games 12
gopher 13
ftp 14
dbus 81
abrt 173
apache 48
ansible 503
```
##### 常见的action

awk中的action可以分为以下5类：

1.表达式语句，包括算术表达式和比较表达式，就是用进行比较和计算的   
2.控制语句，用作进行控制，典型的就是if   else，while等语句，和bash脚本里面用法差不多。  
3.输入语句，用来做为输入，变量赋值就算是。  
4.输出语句，用来输出显示的，典型的是print和printf  
5.组合语句，这个很多理解，就是多种语句的组合  

---

### 控制语句

- if-else语句.

语法:  
> if (condition) {then-body} else {[ else-body ]}

例子:  

```
awk -F: '{if ($1=="root") print $1, "Admin"; else print $1, "Common User"}' /etc/passwd

awk -F: '{if ($1=="root") printf "%-15s: %s\n", $1,"Admin"; else printf "%-15s: %s\n", $1, "Common User"}' /etc/passwd

awk -F: -v sum=0 '{if ($3>=500) sum++}END{print sum}' /etc/passwd
```

- while语句

语法:  

>  while (condition){statement1; statment2; ...}

- 例子,打印每行$1,$2,$3这3个列,

```
[root@localhost ~]$awk -F : '{i=1;while (i<=3) {print $i;i++}}' /etc/passwd

root 1
x 2
0 3
bin 1
x 2
1 3
daemon 1
x 2
2 3
```

- for语句


语法:  

>  for ( variable assignment; condition; iteration process) { statement1, statement2, ...}

- 例子,还是上面while循环的例子

```
[root@localhost ~]$awk -F : '{for(i=1;i<=3;i++) {print $i}}' /etc/passwd
root
x
0
bin
x
1
daemon
x
2
adm
x
3
```

- case语句

语法：

> switch (expression) { case VALUE or /REGEXP/: statement1, statement2,... default: statement1, ...}

- break 和 continue语句

常用于循环或case语句中

- next

提前结束对本行文本的处理，并接着处理下一行；

例如，下面的命令将显示,如果为偶数,则next下一行,只显示其ID号为奇数的用户：

```
awk -F: '{if($3%2==0) next;print $1,$3}' /etc/passwd
bin:x:1:1:bin:/bin:/sbin/nologin1
adm:x:3:4:adm:/var/adm:/sbin/nologin3
sync:x:5:0:sync:/sbin:/bin/sync5
halt:x:7:0:halt:/sbin:/sbin/halt7
operator:x:11:0:operator:/root:/sbin/nologin11
```

### 数组

下面是数组的语法：

> array_name[index]=value

其中，array_name是数组的名称，index是数组索引，value是任意值分配给数组的元素。

要遍历数组中的每一个元素，需要使用如下的特殊结构,其中，var用于引用数组下标，而不是元素值：

> for (var in array) { statement1, ... }

- 例子.列举当前所有TCP连接状态  
每出现一被/^tcp/模式匹配到的行，数组S[$NF]就加1，NF为当前匹配到的行的最后一个字段，此处用其值做为数组S的元素索引；
```
[work@docker ~]$netstat -an | awk '/^tcp/ {++S[$NF]};END{for (a in S) printf "%-15s %d\n",a,S[a]}'
LISTEN          33
ESTABLISHED     101
TIME_WAIT       901

```

> 解释:

>/pattern/ ----模式匹配,这里匹配tcp开头的行  
++S[$字段]---数组,遇到相同字段,加1  
END -----------awk命令结束后,打印  
for () -----------循环,注意循环语句要用()括号  
printf ----------格式化打印..格式:printf "模式 \n",打印项.  "模式"定义完后("%-15s %d\n") 要使用,逗号和后面的打印项分隔

### 经典例子

- 查找请求数请20个IP（常用于查找攻来源）

```
netstat -anlp|grep 80|grep tcp|awk ‘{print $5}’|awk -F: ‘{print $1}’|sort|uniq -c|sort -nr|head -n20
```
- 用tcpdump嗅探80端口的访问看看谁最高

```
tcpdump -i eth0 -tnn dst port 80 -c 1000 | awk -F"." ‘{print $1"."$2"."$3"."$4}’ | sort | uniq -c | sort -nr |head -20
```
- 查找较多time_wait连接

```
netstat -n|grep TIME_WAIT|awk ‘{print $5}’|sort|uniq -c|sort -rn|head -n20
```

- 网站日志分析篇

1.获得访问前10位的ip地址

```
cat access.log|awk ‘{counts[$(11)]+=1}; END {for(url in counts) print counts[url], url}’
```

2.访问次数最多的文件或页面,取前20

```
cat access.log|awk ‘{print $11}’|sort|uniq -c|sort -nr|head -20
```

3.统计网站流量（G)

```
cat access.log |awk ‘{sum+=$10} END {print sum/1024/1024/1024}’
```

4.统计http status

```
cat access.log |awk ‘{counts[$(9)]+=1}; END {for(code in counts) print code, counts[code]}'
```

5.如果日志最后一列记录的是页面文件传输时间，则有列出到客户端最耗时的页面

```
cat access.log |awk ‘($7~/.php/){print $NF " " $1 " " $4 " " $7}’|sort -nr|head -100
```
