## Ansible变量

ansible有多重方式定义变量,还可以通过fact来获取变量.接下来学习一下ansibled 变量知识



## 在playbook中定义变量

1.在playbook中定义vars字段可以定义变量.例如:

```
vars:
  key_files: /etc/nginx/ssl/nginx.key
  cert_files: /etc/nginx/ssl/nginx.crt
  conf_files: /etc/nginx/sites-avaliable/default
```



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



