---
title:  使用ElastAlert+ELK实现日志监控钉钉告警
date: 2020-08-25 22:59:58
tags:  elk
categories: elk
comments: true
copyright: true
---



## 使用ElastAlert+ELK实现日志监控钉钉告警

### 介绍

目前公司使用ELK做日志收集和展示分析.所以想对一些关键日志进行监控告警.比如Nginx的5xx日志,比如php-fpm的Fatal严重错误日志等.通过监控ES的日志数据,然后使用Python调用钉钉接口来实现日志的告警

---

### ElastAlert介绍

ElastAlert是一个开源的工具,用于从Elastisearch中检索数据,并根据匹配模式发出告警.github项目地址如下:https://github.com/Yelp/elastalert

官方文档如下:https://elastalert.readthedocs.io/en/latest/elastalert.html

它支持多种监控模式和告警方式,具体可以查阅Github项目介绍.但是自带的ElastAlert并不支持钉钉告警,在github上有第三方的钉钉python项目.地址如下:https://github.com/xuyaoqiang/elastalert-dingtalk-plugin

第三方的钉钉告警插件并没有艾特相关人员的功能,所以我再此基础上进行了二次开发,增加了这个功能

---

<!--more-->

### ElastAlert安装

> 新版的ElastAlert不支持python2了.所以需要安装Python3环境

* 安装依赖

```
sudo yum -y install python3 python3-devel python3-libs python3-setuptools git gcc
```

如果是Ubuntu系统:

```
sudo apt update
sudo apt -y upgrade
sudo apt install -y python3.6-dev
sudo apt install -y libffi-dev libssl-dev
sudo apt install -y python3-pip
sudo apt install -y python3-venv
```

* 安装elastalert模块

```
pip3 install elastalert  -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com
```

* 克隆ElastAlert项目

```
git clone https://github.com/Yelp/elastalert.git
cp -r elastalert /data/
```

* 安装模块

```
cd /data/elastalert/
pip3 install "setuptools>=11.3"

pip install -r requirements.txt  -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com
```

* 创建ElastAlert的索引

```
sudo elastalert-create-index --index elastalert
```

* 修改ElastAlert的配置文件

```
cp config.yaml.example config.yaml
vim config.yaml

rules_folder: rule  #rule匹配模式的目录,可以自定义一个/data/elastalert路径下的相对目录
run_every:          #ElastAlert多久向Elasticsearch发送一次请求
  minutes: 1
buffer_time:        #如果某些日志源不是实时的，则ElastAlert将缓冲最近一段时间的结果.这个值默认是15,但是无法触发告警,设置为1正常
  minutes: 1
es_host: localhost      #ES集群节点,随便指定任意一台均可
es_port: 9200           #ES端口号
es_username: elastic    # 如果ES使用了X-pack安全验证,则需要配置此项,否则注释
es_password: password   # 同上
writeback_index: elastalert_status  #ElastAlert索引名
alert_time_limit:       #如果告警发送失败,则会在下面时间范围内尝试重新发送
  days: 2
```

* 配置钉钉报警

```
git clone https://github.com/xuyaoqiang/elastalert-dingtalk-plugin.git
cd elastalert-dingtalk-plugin
pip3 install -r requirements.txt -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com
cp -r elastalert_modules /data/elastalert/
```

----

### Rule规则

官方支持很多Rule模式,在`example_rules`目录下也有很多参考Rule可以参考.一般常用的是类型(type)是`frequence`

rule的yaml配置要放在`config.yml`配置文件中定义的目录下,我这里是rule目录.

下面这个rule是监控Nginx的5XX状态码,并且调用钉钉告警

```
#rule名字,必须唯一
name:  the count of servnginx log that reponse status code is 5xx and it appears greater than 5 in the period 1 minute

#类型,官方提供多种类型
type: frequency

#ES索引,支持通配符
index: logstash-*-nginx-access-*

#在timeframe时间内,匹配到多少个结果便告警
num_events: 1

#监控周期.默认是minutes: 1
timeframe:
  seconds: 5  
  
#匹配模式.
filter:
- range:
    status:
      from: 500
      to: 599
      
#告警方式,下面是调用第三方的钉钉告警
alert:
- "elastalert_modules.dingtalk_alert.DingTalkAlerter"

#钉钉的webhook
dingtalk_webhook: "https://oapi.dingtalk.com/robot/send?access_token="  #参考地址,需要自行配置
dingtalk_msgtype: text

#原生的告警信息不友好,自定义告警内容的格式
alert_text: "
域    名: {}\n
调用方式: {}\n
请求链接: {}\n
状 态 码: {}\n
后端服务器: {}\n
数      量: {}
"
alert_text_type: alert_text_only

#告警内容
alert_text_args:
- domain
- request_method
- request
- status
- upstreamaddr
- num_hits
```

测试Rule文件是否正确.在elastalert目录下执行下个命令可以测试某个rule是否正常工作

```
/usr/local/bin/elastalert-test-rule --config config.yaml rule/nginx.yaml
```

这一步可能会有一些报错情况.一般都是扩展模块版本或者依赖关系的问题.比如下面这个问题,就需要执行`pip3 install jira==2.0.0`:

```
Traceback (most recent call last):
  File "/usr/local/bin/elastalert-test-rule", line 11, in <module>
    load_entry_point('elastalert==0.1.20', 'console_scripts', 'elastalert-test-rule')()
  File "/usr/lib/python3.6/site-packages/pkg_resources/__init__.py", line 476, in load_entry_point
    return get_distribution(dist).load_entry_point(group, name)
  File "/usr/lib/python3.6/site-packages/pkg_resources/__init__.py", line 2700, in load_entry_point
    return ep.load()
  File "/usr/lib/python3.6/site-packages/pkg_resources/__init__.py", line 2318, in load
    return self.resolve()
  File "/usr/lib/python3.6/site-packages/pkg_resources/__init__.py", line 2324, in resolve
    module = __import__(self.module_name, fromlist=['__name__'], level=0)
  File "/usr/local/lib/python3.6/site-packages/elastalert/test_rule.py", line 20, in <module>
    import elastalert.config
  File "/usr/local/lib/python3.6/site-packages/elastalert/config.py", line 99
    raise EAException("Could not import module %s: %s" % (module_name, e)), None, sys.exc_info()[2]
                                                                          ^
SyntaxError: invalid syntax

或者下面这个错误

[work@idc-function-elk10 elastalert]$ /usr/local/bin/elastalert-test-rule --config config.yaml rule/nginx.yaml
Traceback (most recent call last):
  File "/usr/lib/python3.6/site-packages/pkg_resources/__init__.py", line 570, in _build_master
    ws.require(__requires__)
  File "/usr/lib/python3.6/site-packages/pkg_resources/__init__.py", line 888, in require
    needed = self.resolve(parse_requirements(requirements))
  File "/usr/lib/python3.6/site-packages/pkg_resources/__init__.py", line 779, in resolve
    raise VersionConflict(dist, req).with_context(dependent_req)
pkg_resources.VersionConflict: (elastalert 0.1.20 (/usr/local/lib/python3.6/site-packages), Requirement.parse('elastalert==0.2.4'))

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/bin/elastalert-test-rule", line 6, in <module>
    from pkg_resources import load_entry_point
  File "/usr/lib/python3.6/site-packages/pkg_resources/__init__.py", line 3095, in <module>
    @_call_aside
  File "/usr/lib/python3.6/site-packages/pkg_resources/__init__.py", line 3079, in _call_aside
    f(*args, **kwargs)
  File "/usr/lib/python3.6/site-packages/pkg_resources/__init__.py", line 3108, in _initialize_master_working_set
    working_set = WorkingSet._build_master()
  File "/usr/lib/python3.6/site-packages/pkg_resources/__init__.py", line 572, in _build_master
    return cls._build_from_requirements(__requires__)
  File "/usr/lib/python3.6/site-packages/pkg_resources/__init__.py", line 585, in _build_from_requirements
    dists = ws.resolve(reqs, Environment())
  File "/usr/lib/python3.6/site-packages/pkg_resources/__init__.py", line 774, in resolve
    raise DistributionNotFound(req, requirers)
pkg_resources.DistributionNotFound: The 'jira>=2.0.0' distribution was not found and is required by elastalert
```

---

### 执行ElastAlert

一切没问题后,就可以执行ElastAlert.如果是针对单个Rule执行就使用下列命令.(在ElastAlert目录下)

```
[root@idc-function-elk10 elastalert]# python3  -m elastalert.elastalert --verbose --rule /data/elastalert/rule/nginx.yaml


1 rules loaded
INFO:elastalert:Starting up
INFO:elastalert:Disabled rules are: []
INFO:elastalert:Sleeping for 59.999906 seconds
INFO:elastalert:Queried rule the count of servnginx log that reponse status code is 5xx and it appears greater than 5 in the period 1 minute from 2020-08-26 09:11 CST to 2020-08-26 09:26 CST: 10000 / 10000 hits (scrolling..)
INFO:elastalert:Queried rule the count of servnginx log that reponse status code is 5xx and it appears greater than 5 in the period 1 minute from 2020-08-26 09:11 CST to 2020-08-26 09:26 CST: 20000 / 10000 hits (scrolling..)
```

等待几秒钟后,钉钉会收到告警(我这里用的是200状态码测试).报警内容是Rule配置文件中自定义的格式和内容

![image-20200826094746820](https://img2.jesse.top/image-20200826094746820.png)

---

### Rule2. 监控php-fpm的Fatal错误信息

fpm的错误日志也收集到了ELK中.我们期望只要pfm日志中出现"Fatal"关键字错误信息就立即告警.最初计划是用ElastAlert的黑名单(blacklist)类型的Rule.但是由于fpm的错误日志没有解析,而是直接保存原始日志,所以不符合要求.

参考github上我提的ISSUE:[balacklist query hits but no matches no alerts](https://github.com/Yelp/elastalert/issues/2937)

也可以用Any类型的type.Rule文件如下:

```
[root@idc-function-elk10 elastalert]# sed '/^#/d' rule/php-fpm.yaml | sed '/^$/d'
name: monitor the fatal,error log in php-fpm log
type: any
index: logstash-*-fpm-error-*
num_events: 1
timeframe:
  seconds: 5
filter:
 - query:
     query_string:
       query: "message: \"PHP Fatal\""  #匹配Fatal关键字
alert:
- "elastalert_modules.dingtalk_alert.DingTalkAlerter"
dingtalk_webhook: "https://oapi.dingtalk.com/robot/send?access_token=" 
dingtalk_msgtype: text
alert_text: "
 主机: {}\n
 IP地址: {}\n
 业务线: {}\n
 日志类型: {}\n
 完整日志: {}
"
alert_text_type: alert_text_only
alert_text_args:
  - host.name
  - host.ip
  - fields.project
  - fields.type
  - message
```

---

### 启动ElastAlert

开启一个Screen然后,使用nohup挂起执行.

```
nohup python3  -m elastalert.elastalert --verbose &
```

---

### 钉钉告警二次开发

当前日志告警只是简单的发送到告警群,由于没有艾特相关人员,所以大家还是无法第一时间看到告警信息,所以需要增加这个功能,大致思路是根据业务线来艾特相关负责人.

但是中台的业务线有些复杂,因为不同的项目负责人不同.所以需要特殊对待.

**准备工作**:

日志告警中必须含有以下几个字段:

* 业务线
* 日志类型
* 如果是Nginx日志,则需要有Nginx的域名

修改原生的钉钉告警的alert动态方法,内容如下

```
import re
   
   def alert(self, matches):
        headers = {
            "Content-Type": "application/json",
            "Accept": "application/json;charset=utf-8"
        }
        #body拿到的是告警内容字符串
        body = self.create_alert_body(matches) 
        #利用正则找到告警日志中的type关键字,也就是日志类别.当前主要有Nginx日志和fpm日志
        res_type = re.findall("type: ([a-z]+)", body)
        #找到业务线,当前有hsq,iqg,msf.bbh,mg等业务线
        res_Project = re.findall(r"业务线: ([a-z]+)", body)
        #如果告警日志没有相关字段,则抛出异常
        if not res_type or not res_Project:
            raise EAException("告警字段中日志类型或者业务线没有配置")
            
        #将正则匹配到的列表类型转换为字符串
        Type = "".join(res_type)
        Project = "".join(res_Project)
        
        #根据相关业务,艾特相关人员
        if Project == "hsq":
            at_list = ['1560xxxxxx']
        elif Project == "iqg":
            at_list = ["137xxxxxx"]
        elif Project == "bbh":
            at_list = ["176xxxxxx"]
        elif Project == "msf":
            at_list = ["180xxxxxx"]
        #如果是中台业务线,并且是Nginx的告警,则需要艾特具体人员
        elif Project == "mg":
            if "nginx" in Type:
            	  #匹配到域名
                mg_Project = re.findall("domain: (.*)\.doweidu\.com", body)[0]
                #如果是交易中台的域名
                if mg_Project == "trade": 
                    at_list = ["177xxxxxx"]
                #如果是消息中台域名
                elif mg_Project == "message.center":
                    at_list = ["170xxxxxx"]
                #如果是商品中台域名
                elif mg_Project == "goods.center":
                    at_list = ["186xxxxxx"]
                #否则艾特中台负责人
                else:
                    at_list = ["186xxxxx"]
            else:
                at_list = ["186xxxxx"]
				
				#为了防止遗漏,如果没有at_list变量,则艾特我本人.使用locals().keys()可以判断某个变量是否被定义
        if (not "at_list" in locals().keys()): at_list = ["17749739691"]
        payload = {
            "msgtype": self.dingtalk_msgtype,
            "text": {
                "content": body
            },
            "at": {
                "atMobiles": at_list,   #艾特相关人员
                "isAtAll":False
            }
        }
        try:
            response = requests.post(self.dingtalk_webhook_url, 
                        data=json.dumps(payload, cls=DateTimeEncoder),
                        headers=headers)
            response.raise_for_status()
        except RequestException as e:
            raise EAException("Error request to Dingtalk: {0}".format(str(e)))
```

