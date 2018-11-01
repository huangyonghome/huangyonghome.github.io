---
title:  ansible--loop循环
date: 2018-08-28 22:59:58
tags:  Ansible
categories: Ansible
comments: true
copyright: true
---

## ansible--loop循环

#### 标准循环

* 下面是一个简单的标准loop循环的例子:

```
[root@localhost playbook]$vim loop-useradd.yaml

---
- hosts: all
  tasks:
    - name: add several users
      user:
         name: "{{ item }}"
         state: present
         groups: "wheel"
      loop:
         - testuser1
         - testuser2
```

<!--more-->

* 如果变量是一个Yaml列表,则可以循环这个列表:

```
loop: "{{ somelist }}"
```

所以上面的例子也可以改成:

```
---
- hosts: all
  tasks:
    - name: add several users
      user:
         name: "{{ item }}"
         state: present
         groups: "wheel"
      loop: [ testuser3,testuser4]

```

循环实际上相当于下面的语句:

```
- name: add user testuser1
  user:
    name: "testuser1"
    state: present
    groups: "wheel"
- name: add user testuser2
  user:
    name: "testuser2"
    state: present
    groups: "wheel"
```



* 也可以循环一个字典,例如:

```
- name: add several users
  user:
    name: "{{ item.name }}"
    state: present
    groups: "{{ item.groups }}"
  loop:
    - { name: 'testuser1', groups: 'wheel' }
    - { name: 'testuser2', groups: 'root' }
```

---

#### loop与yum,apt

有一些模块有自带列表循环功能,这比使用loop循环更好,例如:

```
[root@localhost playbook]$vim loop-yum.yaml

---
- hosts: all
  tasks:
   - name: optimal yum
     yum:
       name: [httpd,nginx,iotop]
       state: present
```

这比使用下面的loop循环语句要更好,因为loop循环可能会有Yum安装软件的依赖问题

```
- name: non optimal yum, not only slower but might cause issues with interdependencies
  yum:
    name: "{{item}}"
    state: present
  loop: "{{list_of_packages}}"
```

---

#### with_items迭代

ansible默认使用item作为循环迭代变量名,with_items列出了一个items的列表.迭代的传递值给{{item}}变量.例如:

```
-name: install apt packages
 apt: pkg={{item}} update_cache=yes cache_valid_time=3600
 sudo: True
 with_items:
    - git
    - libjpeg-dev
    - libpq-dev
    - memcached
    - nginx
    - postgresql
```

> 在apt,yum等模块中,使用with_items语句安装软件包效率更高,这是因为ansible会将整个软件包的列表一起传递给apt,yum模块,相当于只调用一次apt,yum命令

----

#### 嵌套循环

如果一个模块要使用2个循环那该怎么办.在前一个循环的基础上,再进行一次循环时,就可以使用嵌套循环.

例如:

```
     - name: create project dir
       file: path=/data/apps/msf-{{ item[0] }}-api-{{ dwd_env }}/{{ item[1] }} recurse=yes owner=root group=root mode=0644 state=directory
       with_nested:
         - ['open','internal']
         - ['releases','vendor','shared']
```

  item[0]表示循环'open','internal'.在每次循环item[0]时都循环一遍item[1],也就是'release,vendor,shared.
   上面的循环语句其实类似于shell的命令:
   mkdir -pv /data/apps/msf-{open,internal}-api/{releases,vendor,shared}
   
---

### 列表循环

有时候循环不适合写死变量.因为万一变量没有值的话,就需要改写playbook里的task.这是非常不方便的.

比如:如果下面的openapi变量没有值,或者如果不需要新建这个目录,就需要修改这个task.

```
file: path={{ nginx_site_dir }}/{{ project_name }}-{{ item[0] }}-{{ dwd_env }}/{{ item[1] }}/vendor recurse=yes owner=work group=work mode=0775 state=directory
with_nested:
      - [ "{{ openapi }}"," {{ xxx }},"{{ haha }}""]
      - ['releases','shared']
      
```
 
此时列表循环就非常有帮助.例如:
 
```
vars:
   api:
     - openapi
     - internal
     - script
     
tasks:

   - name: create dev beta project dir
     file: path=/tmp/hsq-{{ item[0] }}/{{ item[1] }} recurse=yes owner=work group=work mode=0775 state=directory
     with_nested:
         - "{{ api }}"
         - ['releases','shared']
              
   
```
 				
 		
 
 这样一来只需要修改api的变量,而不需要修改playbook的task.
 
 执行结果如下:
 
 
```
TASK [create dev beta project dir] *********************************************************************************************************************************************
changed: [docker] => (item=[u'openapi', u'releases'])
changed: [docker] => (item=[u'openapi', u'shared'])
changed: [docker] => (item=[u'internal', u'releases'])
changed: [docker] => (item=[u'internal', u'shared'])
changed: [docker] => (item=[u'script', u'releases'])
changed: [docker] => (item=[u'script', u'shared'])

PLAY RECAP *********************************************************************************************************************************************************************
docker                     : ok=2    changed=1    unreachable=0    failed=0

```
