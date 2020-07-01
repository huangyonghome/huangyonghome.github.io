---
title: acme.sh自动申请SSL证书脚本  
date: 2020-06-30 00:44:48  
tags: scripts
categories: [Linux-Basic,shell&shell脚本 ]  
comments: true  
copyright: true  
---



## acme.sh自动申请SSL证书脚本

acme.sh是GitHub上的一个项目.有关这个工具的介绍可以参考github,或者查看Linux-证书目录下的相关笔记


```
#!/bin/bash
#description: 使用acme.sh工具通过自动DNS验证方式申请SSL证书.
#date: 20190620

#判断是否有指定域名,以及域名
if [ $# != 2 ];then
   echo "usage: $0 domain_name ssl_install_dir"
   exit 1
fi
#接收要申请SSL证书的域名参数
ssl_domain=$1
#接收证书安装目标路径的参数
ssl_install_dir=$2

#判断申请的是否是通配符证书
if `echo $ssl_domain | grep "^\*" > /dev/null 2>&1`;then

    #如果是通配符证书,那么替换到域名前面的"*."
    ssl_name=$(echo $ssl_domain | sed 's@\*\.@@')

else

    ssl_name=$ssl_domain

fi

#检查证书安装目录,判断该域名是否已经申请过证书.如果已经申请过,则直接退出

#新建ssl_dir证书路径变量
ssl_dir=${ssl_install_dir}/${ssl_name}

if [ -d $ssl_dir ];then

    echo "$ssl_domain certificate has already installed" && exit 0

fi

#导入环境变量
export DP_Id=DNS服务器的AccessKeyID
export DP_Key=DNS服务器的Secret

#申请证书
cd /home/work/.acme.sh/

# 判断申请的是通配符证书,还是单域名证书
if [ $ssl_domain == $ssl_name ];then

	./acme.sh --issue --dns dns_dp  -d $ssl_name

else
	./acme.sh --issue --dns dns_dp  -d $ssl_name -d $ssl_domain

fi

#判断证书申请是否成功
if [ $? != 0 ];then
     echo "$ssl_domain certificate applied failed" && exit 1
else
    echo "$ssl_domain certificate applied successfully"
fi


#新建证书目标路径
mkdir -p $ssl_dir

#安装证书
./acme.sh  --installcert  -d  $ssl_name  \
       --key-file ${ssl_dir}/${ssl_name}.key  \
       --fullchain-file ${ssl_dir}/fullchain.cer     \
       --reloadcmd  "supervisorctl -c /etc/supervisord/supervisord.conf restart nginx"
```