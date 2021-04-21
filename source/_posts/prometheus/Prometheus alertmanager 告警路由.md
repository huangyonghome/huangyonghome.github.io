---
title: Prometheus Alertmanager 告警路由
date: 2021-01-10 07:59:58
tags: prometheus
categories: prometheus
comments: true
copyright: true
---



## Prometheus Alertmanager 告警路由

### 介绍

公司目前Prometheus监控了IDC数据中心的主机,中间,站点等,同时也监控了阿里云线上的rabbitmq,mysql,kong(所有资源都是ECS自己搭建的,非阿里云的saas服务)

prometheus使用了第三方的钉钉监控插件(prometheus-webhook-dingtalk),github地址: https://github.com/timonwong/prometheus-webhook-dingtalk 

Prometheus通过alertmanager将告警信息发送到钉钉机器人

---

### 告警路由

我们需要将阿里云的线上中间件监控告警发送到阿里云的钉钉群.IDC资源的监控告警发送到IDC的钉钉群,不同的钉钉群面对的人群也不同.方便监控告警信息的分类和管理.

幸好,alertmanager天生支持告警路由的功能,将不同的告警信息发送给不同的receiver接收人

---

### alertmanager概念介绍



Prometheus本身并不提供告警功能,所有告警信息都是发送给Alertmanager处理.Alertmanager接收到告警信息后负责将它们分组,抑制,静默,然后路由到相关接收者.

##### Grouping分组

分组功能将多个同一类型的告警合并一起后发送,这在某个服务发生故障从而影响其他几十,上百个相关依赖性的服务时非常有用,可以有效避免告警信息轰炸.例如当网络出现问题时,可能该网络下的数百个服务都出现访问故障,结果数以百计的告警被发送给Alertmanager.此时Alertmanager将同类型的服务合并到一起仅仅使用单条告警通知发送给接收者

<!--more-->

##### Inhibition抑制

抑止是指如果某个告警已经触发,那么抑止其他有关该服务的告警消息.

例如如果某个集群A不可达,已经触发了告警.那么其他B,C,D等集群和服务发出的A集群不可达的告警通知将被Alertmanager抑止.告警抑制机制可以防止数百上千的重复故障告警



##### Silences静默

Silence静默配置的作用类似于Zabbix中的Maintenance维护功能，可以配置一个时间区间和相关规则，符合该配置的事件将不会进行告警。比如明确凌晨会暂停服务，这个时候就可以提前设置好静默规则，减少不必要的告警骚扰。Prometheus的Silence规则只需要通过AlertManager的Web界面就可以完成，不需要配置文件

---

### Alertmanager主要处理流程

处理流程:
**1.** 接收到Alert，根据labels判断属于哪些Route（可存在多个Route，一个Route有多个Group，一个Group有多个Alert）。
**2.** 将Alert分配到Group中，没有则新建Group。
**3.** 新的Group等待group_wait指定的时间（等待时可能收到同一Group的Alert），根据resolve_timeout判断Alert是否解决，然后发送通知。
**4.** 已有的Group等待group_interval指定的时间，判断Alert是否解决，当上次发送通知到现在的间隔大于repeat_interval或者Group有更新时会发送通知。

---



### Alertmanager配置文件

Alertmanager可以通过命令行配置和yaml配置文件配置.`./alertmanager -h` 可以打印出所有的命令行配置选项.这里主要介绍`alertmanager.yaml`这个配置文件的相关配置

`alertmanager.yaml` 配置文件主要字段有如下几个:

```
global:
   #顶级路由,在route下可以定义routes子路由树
   route:
   
   #告警接收者,如果有多个routes,那么需要在receivers下定义多个接收者
   receivers:
   
   #告警抑制配置
   inhibit_rules:
```



##### route

`route` 字段定义路由树的节点,以及子节点的相关配置.子节点可以从父节点继承所有配置参数.

每个告警进入到顶级配置的route.该route必须是一个默认路由,匹配所有Prometheus的告警规则.然后去遍历所有子路由.如果`continue` 设置为`false` ,在匹配到第一个子路由(routes)后就停止继续匹配,并且交给子路由的receiver发出告警.如果`continue` 设置为`true` 则继续与后续的其他子路由(routes)匹配.如果某条告警信息不匹配任何子路由,或者当前没有配置任何子路由,则交给默认的顶级route处理.

下面是一个route路由配置的案例

```
route:
  resolve_timeout: 5m #一条告警消息发出后,如果在多长时间内没有再次告警,则认为该故障已经解除,发送告警恢复消息
  group_by: ['alertname']  #根据告警规则名称分组 也就是rule文件的顶部定义的名称.还可以根据标签label或者job来分组
  group_wait: 30s #一组警报消息到达后等待多少秒发送,这允许在这期间收集更多同组警报消息
  group_interval: 5m #发送某组的第一个警告信息后,等待多久继续发送改组新的警告消息
  repeat_interval: 4h #如果告警信息已经成功发送,等待多久重新发送
  receiver: 'default' #顶级route的默认接收者,所有未匹配到子路由的消息都发送到该receiver
  routes: #子路由配置.在子路由下默认继承上面的group_by,group_wait等配置,也可以重写
    - receiver: "aliyun" #定义子路由的接收者
      match_re: #匹配告警信息,可以通过Match固定匹配,也可以通过match_re正则匹配
        service: rabbitmq|mysql #只要是service这个lable标签,值为rabbitmq或者mysql的告警信息都使用该子路由处理
    - receiver: 'frontend-pager'
    group_by: [product, environment] #对标签名为product和environment的告警信息作为一组
    match:
      team: frontend 
```



##### receiver

该配置定义了一个或者多个告警消息接收器.Alertmanager并不会主动联系receiver,而是需要第三方webhook插件实现告警接收.我们这里使用的是钉钉告警插件.关于钉钉告警插件的配置在后文会有详细介绍.

有多少个子route,就对应多少个receiver.(当然也可以多个子route对应同一个receiver).receiver定义了告警消息接受地址

下面是receiver的配置,定义了2个reciever,对应上面的route.`aliyun`的receiver用来接收rabbitmq,mysql的告警通知,`defualt` 用来接收其他所有的告警消息

```
      
# 定义接收者信息 
receivers:
- name: 'default'
  webhook_configs: #第三方插件的webhook地址
  - url: 'http://localhost:8060/dingtalk/default/send' #这里使用的是我本地运行的钉钉插件的发送告警消息
- name: 'aliyun'
  webhook_configs:
  - url: 'http://localhost:8060/dingtalk/aliyun/send'
```



##### inhibit_rules

在Alertmanager配置文件中,使用`inhibit_rules` 定义一组告警的抑制规则.当已经发送的告警通知匹配到target_match和target_match_re规则，当有新的告警规则如果满足source_match或者定义的匹配规则，并且以发送的告警与新产生的告警中equal定义的标签完全相同，则启动抑制机制，新的告警不会发送。

上面这段概念理解起来比较拗口,使用下面的配置作为一个案例解读:

```
- source_match:
    alertname: NodeDown
  target_match_reg:
    severity: ~"middle|low"
  equal:
    - node
```

当接收到一个lable名称为`alertname` ,值为`NodeDown` 的告警.并且为该告警发送了一个通知:

```
{alertname="NodeDown",node="x.x.x.x",...} time annotation
```

那么Alertmanager就会创建一条抑制规则:

```
{node="x.x.x.x",serverity=~"middle|low"}
```

如果新的告警满足severity=~"middle|low",并且node标签相等(也就是equal的作用).那么该告警就会被抑制..例如该主机上的mysqldown的告警消息就不会被发送

```
{alertname="MysqlDown",node="x.x.x.x",serverity="middle",...} time annotation
```

这也是我们期望看到的,因为当我们收到了某个主机节点Down的告警通知,那么该主机上的所有服务不可用的告警消息不应该再次发送.

---



### 钉钉插件

目前使用的是prometheus-webhook-dingtalk钉钉告警插件.在github上可以直接下载二进制文件运行.默认监听在8060端口.使用`./prometheus-webhook-dingtalk -h`可以指定自定义配置文件和监听端口

该插件提供了一下路由供Alertmanager的webhook_configs使用

`/dingtalk/<profile>/send` 

这里的profile需要在插件启动时`-ding.profile` 中指定相应的名称.为了支持多个receiver,同时往多个钉钉自定义机器人发送告警消息,该插件可以指定多个`-ding.profile` 参数,从而指定多个钉钉机器人的地址.例如下面的prometheus-webhook-dingtalk启动配置文件:

```
[work@172 prometheus-webhook-dingtalk]$ systemctl cat prometheus-webhook-dingtalk
# /usr/lib/systemd/system/prometheus-webhook-dingtalk.service
[Unit]
Description=prometheus-webhook-dingtalk
After=network-online.target

[Service]
Restart=on-failure
user=root
#定义default和aliyun2个钉钉机器人地址,用来实现告警路由,将不同的告警消息发送到不同的钉钉群
ExecStart=/data/prometheus-webhook-dingtalk/prometheus-webhook-dingtalk \
          --ding.profile=default=https://oapi.dingtalk.com/robot/send?access_token=c35575a65532bc15b0ff50cfad1xxxxx \
          --ding.profile=aliyun=https://oapi.dingtalk.com/robot/send?access_token=f064adad17bc66ad30d94c7c9cxxxxxxx

[Install]
WantedBy=multi-user.target
```

上面定义了2个profile:`aliyun`和`default` 对应了`alertmanager.yaml`配置文件中的不同的receiver配置.需要注意的是不同的profile,它供alertmanager调用的地址也是不同的.比如:

```
default的profile的地址为: http://localhost:8060/dingtalk/default/send
```

经过测试下来,不同的监控对象告警信息发送到不同的钉钉群组,方便相关的团队和人员第一时间接收和处理

---

### 参考资料

https://prometheus.io/docs/alerting/latest/configuration/   #alertmanager 官方介绍

https://www.kancloud.cn/pshizhsysu/prometheus/1803807 #Alertmanager介绍

https://www.pianshen.com/article/410280293/ #钉钉插件介绍



