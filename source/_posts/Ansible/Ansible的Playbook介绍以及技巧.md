---
title:  Ansible-Playbook
date: 2018-08-26 22:59:58
tags:  Ansible
categories: Ansible
comments: true
copyright: true
---

## Ansible的Playbook介绍以及技巧

playbook的格式是YAML.以下是一个playbook的范例:

<!--more-->
```
---
- hosts: webservers
  vars:
    http_port: 80
    max_clients: 200
  remote_user: root
  tasks:
  - name: ensure apache is at the latest version
    yum: pkg=httpd state=latest
  - name: write the apache config file
    template: src=/srv/httpd.j2 dest=/etc/httpd.conf
    notify:
    - restart apache
  - name: ensure apache is running
    service: name=httpd state=started
  handlers:
    - name: restart apache
      service: name=httpd state=restarted
```

通过上面的范例可以总结出playbook的基础语法和注意点:

* playbook的最顶部以---3个横杠开头
* 相同元素以-减号排列,在缩进中空格的数量不重要,但是相同阶层的元素要左对齐(不能使用tab字符)
* 可以不使用双引号来括住字符串
* name是任务的描述,不是必需.但是为了playbook的易读性,建议name要有.一个name只能包含一个task.



#### playbook的使用技巧

* remote_user表示playbook允许时以哪个用户运行,如果加上sudo:yes.表示以sudo方式运行:

  ```
  - hosts: webservers
    remote_user: yourname
    sudo: yes
  ```

  如果是仅仅在一个task中使用sudo.而不是在整个playbook中使用:

  ```
  ---
  - hosts: webservers
    remote_user: yourname
    tasks:
      - service: name=nginx state=started
        sudo: yes
  ```



* k/v键值对(模块和命令)可以在写在一行,用:冒号隔开,也可以用> 符号换行:

  ```
  - name: test
    command: > 
    ls /home/
  ```

  如果命令行太长,可以用空格或者缩进换行

  ```
  tasks:
    - name: Copy ansible inventory file to client
      copy: src=/etc/ansible/hosts dest=/etc/ansible/hosts
              owner=root group=root mode=0644
  ```

* 命令可以用变量,变量的引用方式为{{ 变量名 }}

  ```
  tasks:
    - name: create a virtual host file for {{ vhost }}
      template: src=somefile.j2 dest=/etc/httpd/conf.d/{{ vhost }}
  ```



* 忽略错误,继续执行下一个task..默认情况下playbook如果执行命令失败就会终止整个Playbook的执行.使用ignore_erros参数可以在执行某个task失败时,自动忽略继续执行下一个task:

  ```
  tasks:
    - name: run this command and ignore the result
      shell: /usr/bin/somecommand
      ignore_errors: True
  ```

  或者采用下列方式.如果somecommand失败,则执行某个一定为true的命令:

  ```
  tasks:
    - name: run this command and ignore the result
      shell: /usr/bin/somecommand || /bin/true
  ```

* limit参数 限定执行的主机范围.例如下面的例子限定在centos7这台主机上执行.当然也可以限定在一个组内的主机上执行

  ```
  [root@ansible ansible]$ansible-playbook test.yaml --limit centos7
  ```

* --list-hosts参数查看一个Playbook影响的所有主机.例如下列命令可以看到Playbook将在哪些主机上执行:

  ```
  [root@ansible ansible]$ansible-playbook test.yaml --list-hosts
  
  playbook: test.yaml
  
    play #1 (all): all	TAGS: []
      pattern: ['all']
      hosts (3):
        centos6
        centos7
        web
  ```

* -- check 检测模式,playbook定义的任务将在每台远程主机上进行检测,但是并不真正执行:

  ```
  [root@ansible ansible]$ansible-playbook test.yaml --check
  ```

> list--tasks有相似的作用,它列出了playbook的task任务安排:
>
> ```
> [root@localhost playbook]$ansible-playbook --list-tasks nginx.yaml
>  [WARNING]: Could not match supplied host pattern, ignoring: 10.0.4.240
> 
> 
> playbook: nginx.yaml
> 
>   play #1 (10.0.4.240): 10.0.4.240	TAGS: []
>     tasks:
>       check if there is nginx repo on server	TAGS: []
>       install nginx yum repo,if there is no nginx repo	TAGS: []
>       install nginx	TAGS: []
>       create nginx document dir	TAGS: []
>       create nginx log dir	TAGS: []
>       copy nginx configure file	TAGS: []
>       copy nginx web file	TAGS: []
>       start nginx	TAGS: []
> 
> ```



* --forks=数字.指定并发执行的任务数.默认是5.调高这个值可以加快ansible执行效率

* 显示命令的执行返回结果.Asible默认并不会显示命令的执行结果,通过debug模块可以输出命令的执行结果.例如:

```

---
  - hosts: all
    remote_user: root
    tasks:
      - name: get variables
        shell: cat /etc/redhat-release
        register: foo
      - name: print variable
        debug: msg="the vairables is {{ foo.stdout }}"
```

执行结果:

```
[root@ansible ansible]$ansible-playbook test.yaml --limit centos7

PLAY [all] **********************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************
ok: [centos7]

TASK [get variables] ************************************************************************************************************************
changed: [centos7]

TASK [print variable] ***********************************************************************************************************************
ok: [centos7] => {
    "msg": "the vairables is CentOS Linux release 7.2.1511 (Core) "
}

PLAY RECAP **********************************************************************************************************************************
centos7                    : ok=3    changed=1    unreachable=0    failed=0   

```

或者使用下面的语句.结果是一样的

```
---
  - hosts: all
    remote_user: root
    tasks:
      - name: get variables
        shell: cat /etc/redhat-release
        register: foo
      - name: print variable
        #debug: msg="the vairables is {{ foo.stdout }}"
        debug: var=foo.stdout
```

执行结果:

```
[root@ansible ansible]$ansible-playbook test.yaml --limit centos7

PLAY [all] **********************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************
ok: [centos7]

TASK [get variables] ************************************************************************************************************************
changed: [centos7]

TASK [print variable] ***********************************************************************************************************************
ok: [centos7] => {
    "foo.stdout": "CentOS Linux release 7.2.1511 (Core) "
}

PLAY RECAP **********************************************************************************************************************************
centos7                    : ok=3    changed=1    unreachable=0    failed=0 
```

