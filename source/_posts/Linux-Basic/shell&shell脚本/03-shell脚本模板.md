---
title: shell脚本模板
date: 2018-06-12 22:59:58
tags:  Linux
categories: [Linux-Basic,shell&shell脚本 ]
comments: true
copyright: true
---


## shell脚本模板

### 介绍

为了保持良好的脚本编写习惯,在编制一个脚本时,需要有以下参数:

- 脚本用途描述  
- 脚本作者  
- 脚本版本
- 脚本更新记录  
- 脚本编写时间

---

下面是一个常见的脚本头部内容:

```
#!/bin/bash
#Description: this is the standard format for the head part of scripts
#Author: Jesse
#Version:0.1
#Datetime:2016-10-31
#Name:script.sh
```
---
<!--more-->

##### 自定义一个脚本模板文件:

```
[root@localhost ~]$cat template.sh 
if ! grep "[^[:space:]]" $1 &>/dev/null; then
cat > $1 << EOF
#!/bin/bash
#Description:
#Author: Jesse
#Version:0.1
#Datetime:`date +%F`
#Name:`basename $1`
export PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin
EOF

fi

vim + $1

```

执行这个模板脚本.附带一个参数,参数名为新脚本名.此时自动会产生一个firstscript.sh脚本,而且已经帮我们编写好了脚本头部内容,且自动使用vim编辑器进入编辑界面

```
[root@localhost ~]$bash template.sh firstscript.sh

#!/bin/bash
#Description:
#Author: Jesse
#Version:0.1
#Datetime:2017-10-24
#Name:firstscript.sh
export PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin
```

