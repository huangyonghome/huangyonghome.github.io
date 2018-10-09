---
title:  Ansible-Roles
date: 2018-08-27 22:59:58
tags:  Ansible
categories: Ansible
comments: true
copyright: true
---

## Ansible-Roles

  对于中小型项目来说playbook结合include就足以胜任,清晰,有效的完成自动化部署工作了.但是如果是对于大型的项目,几十个playbook来说,可能会造成文件繁多,目录结构不清晰,命名不规范,以及后期维护成本大大升高.

ansible的roles功能就是为了解决这个问题应运而生.roles字面意思是"角色",可以理解为各个不同模块功能的关联集合.roles主要包括以下功能模块:var_files,tasks,handlers,templates,files,等等

但是需要注意的是,为了规范和维护期间.roles应该定义一个清晰,明确的目录结构以及文件名.不可随意更改.

---

#### roles目录结构

常见的roles任务包含下列目录结构:

```
playbook_role.yml #role的playbook入口
roles/
  role_name/
      tasks/
      files/
      templates/
      handlers/
      vars/
   role_name2/
      tasks/
      files/
      templates/
      handlers/
      vars/
```

关于roles的目录结构有以下注意事项:

* 一个role必须包含tasks目录.
* 以上的目录结构各代表不同的功能组件,各司其职,不可随意更改目录名.
* 每个目录下需要创建main.yml文件,作为改功能组件的入口.有点类似于Python的init.py文件

下面解释了每个目录的功能含义:

* tasks: 包含一系列的Playbook文件,也就是roles要执行的一系列具体tasks
* handlers: 包含inotify需要执行的handlers任务
* defaults: roles的默认变量
* vars: 变量文件
* files: 文件
* templates: 模板文件
* meta: 元数据

---

#### main.yml文件

main.yml文件是各目录代表的功能模块的入口,main.yml文件可以通过include或者import_task选项导入同目录下的其他yml文件(也就是其他playbook).例如下面的tasks/main.yml文件:

```
#下面是main.yml文件内容.main.yml文件作为tasks的入口,导入了同目录下的redhat和debian2个playbook
# roles/example/tasks/main.yml
- name: added in 2.4, previously you used 'include'
  import_tasks: redhat.yml
  when: ansible_os_platform|lower == 'redhat'
- import_tasks: debian.yml
  when: ansible_os_platform|lower == 'debian'


#下面是tasks/redhat.yml文件,其实就是个独立的playbook
# roles/example/tasks/redhat.yml
- yum:
    name: "httpd"
    state: present

#同理,下面是debian.yml文件
# roles/example/tasks/debian.yml
- apt:
    name: "apache2"
    state: present
```

### 运行roles

运行roles非常简单,只需要在最外层的Playbook上使用roles关键字,指定运行哪些roles目录即可:

```
---
- hosts: webservers
  roles:
     - common
     - webservers
```

一旦运行这个playbook,那么针对上面这2个roles(common和webservers),都会自动执行下面文件:

* roles/\{common,webservers\}/tasks/main.yml (如果该文件存在)
* roles/\{common,webservers\}/vars/main.yml .(如果该文件存在)
* roles/\{common,webservers\}/default/main.yml (如果该文件存在)
* roles/\{common,webservers\}/handlers/main.yml  (如果该文件存在,且tasks目录下的playbook调用了handlers)
* .......

---

