---
title:  Ansible变量
date: 2018-08-26 22:59:58
tags:  Ansible
categories: Ansible
comments: true
copyright: true
---

## Ansible变量

ansible有多重方式定义变量,还可以通过fact来获取变量.接下来学习一下ansibled 变量知识

---
<!--more-->

### 在Inventory主机文件中定义变量

可以对每台主机分配具体的变量,然后在playbook中调用.例如下面的主机Host和host2的变量定义:

```
[atlanta]
host1 http_port=80 maxRequestsPerChild=808
host2 http_port=303 maxRequestsPerChild=909
```

也可以定义属于整个组的变量,这些变量在组内的所有服务器节点上生效:

```
[atlanta]
host1
host2

[atlanta:vars]
ntp_server=ntp.atlanta.example.com
proxy=proxy.atlanta.example.com
```

ansible还会从/etc/ansible/host_vars目录下寻找主机名命名的变量文件.例如如果inventory下有个host1主机,则可以在/etc/ansible/host_vars目录下创建host1文件.并且定义如下内容:

```
---
foo: bar
baz: qux
```

则以上2个变量则会应用到host1主机中.



同理,可以在/etc/ansible/group_vars目录下创建主机组命名的变量文件,比如新建一个atlanta文件,且定义上面变量,则这2个变量会被应用到整个atlanta主机组



---

## 在playbook中定义变量

1.在playbook中定义vars字段可以定义变量.例如:

```
vars:
  key_files: /etc/nginx/ssl/nginx.key
  cert_files: /etc/nginx/ssl/nginx.crt
  conf_files: /etc/nginx/sites-avaliable/default
```

> note: 在Inventory文件中定义变量是用var=value,而在Playbook以及下面讲到的yaml格式的变量文件中变量格式为var: value



2.还可以通过vars_files字段定义一个文件.然后将变量写入到这个文件内:

```
vars_files:
   - nginx.yml
   
nginx.yml文件内容如下:
  key_files: /etc/nginx/ssl/nginx.key
  cert_files: /etc/nginx/ssl/nginx.crt
  conf_files: /etc/nginx/sites-avaliable/default
```

---

### 引用变量

ansible使用jinjia2模板系统在playbook中引用变量.引用方法为{{ varname }} .例如:

```
My amp goes to {{ max_amp_value }}
```



### 

### 查看变量的值

在之前的文档中提到了可以用debug模块来打印变量:

```
- name: print the variable
  debug: var=myvarname
```

---



### 注册变量

经常需要基于task执行的结果来设置变量值.想要实现这个操作可以再调用模块时使用register语句来注册变量

例如,将whiami命令执行的结果保存到变量login中:

```
- name: capture output of whoami command
  command: whoami
  register: login
```



访问login变量中的内容可以使用下列方法:

```
- debug: msg="login is {{login.stdout}}
```

> 如果变量中包含字典,可以使用点号. 或者中括号[]来访问字典内的Key.上面也可以写成:
>
> login['stdout']
>
> 比如还有下面的常见嵌套变量:
>
> ansible_eth1['ipv4']\['address'] 
>
> or:
>
> ansible_eth1['ipv4']\['address']
>
> ansible_eth1.ipv4['address']
>
> ansible_eth1.ipv4.address



### fact

ansible在执行playbook的时候,第一步就是采集主机的fact信息.主要包括:操作系统,IP地址,CPU,内存等等.这些信息都保存在fact变量中.

例如:

```
[root@localhost playbook]$vim facttest.yml

---
 - hosts: all
   remote_user: root
   gather_facts: True
   tasks:
     - name: print the fact varibale:ansible_distrubution
       debug: var=ansible_distribution
```

输出结果如下:

```
[root@localhost playbook]$ansible-playbook facttest.yml

PLAY [all] *******************************************************************************************************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************************************************************************************
ok: [10.0.4.231]
ok: [10.0.4.230]

TASK [print the fact varibale:ansible_distrubution] **************************************************************************************************************************************************************
ok: [10.0.4.231] => {
    "ansible_distribution": "CentOS"
}
ok: [10.0.4.230] => {
    "ansible_distribution": "CentOS"
}

PLAY RECAP *******************************************************************************************************************************************************************************************************
10.0.4.230                 : ok=2    changed=0    unreachable=0    failed=0
10.0.4.231                 : ok=2    changed=0    unreachable=0    failed=0
```

> 在调用fact变量时,并不需要register关键字来注册变量,因为fact是自动注册的.另外还有一些模块也会自带返回fact变量.比如docker模块



#### 查询某台服务器的所有fact信息:

setup特殊模块可以显示fact信息:

```
[root@localhost playbook]$ansible 10.0.4.230 -m setup
10.0.4.230 | SUCCESS => {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "10.0.4.230"
        ],
        "ansible_all_ipv6_addresses": [
            "fe80::20c:29ff:fedc:b1c7"
        ],
        "ansible_apparmor": {
            "status": "disabled"
        },
其余的fact内容忽略
```

由于fact内容实在太多.setup模块还支持filter参数,来匹配想要查找的fact信息.如果没有匹配到则返回空:

```
[root@localhost playbook]$ansible 10.0.4.230 -m setup -a 'filter=ansible_eth*'
10.0.4.230 | SUCCESS => {
    "ansible_facts": {},
    "changed": false
}
[root@localhost playbook]$ansible 10.0.4.230 -m setup -a 'filter=ansible_ip*'
10.0.4.230 | SUCCESS => {
    "ansible_facts": {},
    "changed": false
}
[root@localhost playbook]$ansible 10.0.4.230 -m setup -a 'filter=ansible_*ipv4*'
10.0.4.230 | SUCCESS => {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "10.0.4.230"
        ],
        "ansible_default_ipv4": {
            "address": "10.0.4.230",
            "alias": "eno16777736",
            "broadcast": "10.0.4.255",
            "gateway": "10.0.4.254",
            "interface": "eno16777736",
            "macaddress": "00:0c:29:dc:b1:c7",
            "mtu": 1500,
            "netmask": "255.255.255.0",
            "network": "10.0.4.0",
            "type": "ether"
        }
    },
    "changed": false
}
```

---

### 内置变量

Ansible会自动提供一些变量,即使你没有去定义他们.这些变量是ansible预留的.所以用户不应该手动定义重名的变量.比如下面这些:

* **hostvars**

hostvars可以让你访问所有主机节点的fact信息.如果一台服务器想要访问另一个节点服务器的fact信息,这很有用.一般语法是:hostvars['inventory_hostname']\['fact信息']:例如下面的例子

```
[root@localhost playbook]$vim facttest.yml

---
 - hosts: all
   remote_user: root
   gather_facts: True
   tasks:
     - name: print the fact varibale:ansible_distrubution
       debug: var=hostvars[inventory_hostname]['ansible_all_ipv4_addresses']

```

执行结果:

```
[root@localhost playbook]$ansible-playbook facttest.yml


TASK [print the fact varibale:ansible_distrubution] **************************************************************************************************************************************************************
ok: [10.0.4.231] => {
    "hostvars[inventory_hostname]['ansible_all_ipv4_addresses']": [
        "10.0.4.231"
    ]
}
ok: [10.0.4.230] => {
    "hostvars[inventory_hostname]['ansible_all_ipv4_addresses']": [
        "10.0.4.230"
    ]
}

```



* **inventory_hostname**

inventory_hostname是ansible识别的当前主机的主机名.如果是在hosts文件中定义过别名.比如:

```
beta ansible_ssh_host=10.0.0.250
```

那么inventory_hostname就是beta



* **groups**

当要访问一组主机的变量时,groups变量会很有用.比如一个模板文件需要知道一个test群组内所有服务器的IP地址.那么可以编辑这样一个模板配置文件

```
{% for host in group.test %}
   server {{ host.inventory_hostname }} {{host.ansible_default_ipv4.address}}:80
 {% endfor %}
 
 最终生成的配置文件如下:
 server test1 10.0.4.230
 server test2 10.0.4.231
```

----

### 命令行输入变量

通过向ansible-playbook传递-e var=value参数可以像shell脚本那样使用Playbook.并且该参数的变量拥有最高优先级.例如下面的例子:

```
[root@localhost playbook]$vim greet.yml

---
  - hosts: 10.0.4.230
    vars:
       greeting: "hello"
    tasks:
       - name: print greeting
         debug: msg="{{ greeting }}"
```



通过-e指定greeting变量执行playbook:

```
[root@localhost playbook]$ansible-playbook greet.yml -e 'greeting="hello world"'

TASK [print greeting] ******************************************************************************************************************************
ok: [10.0.4.230] => {
    "msg": "hello world"
}

```

此时greeting的变量值变成了"hello world" .而不是playbook中定义的"hello"

注意,如果变量包含空格,要把整个-e后面的参数用单引号括起来..否则就会得出意外的结果:

```
[root@localhost playbook]$ansible-playbook greet.yml -e greeting="hello world"


TASK [print greeting] ******************************************************************************************************************************
ok: [10.0.4.230] => {
    "msg": "hello"
}
```

另外,-e选项不仅仅可以传递单个字符串,还能传递进一个包含变量的文件.比如下面的例子中,将greetvars.yml这整个文件传递进playbook:

```
[root@localhost playbook]$ansible-playbook greet.yml -e @greetvars.yml
```

---

### 变量优先级:

1.命令行中使用-e参数手动指定

2.inventory主机文件,或者yaml文件定义的主机变量或者群组变量

3.FACT变量

4.role的default/main.yml文件定义的变量

