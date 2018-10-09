---
title:  Ansible playbook高级特性
date: 2018-08-26 22:59:58
tags:  Ansible
categories: Ansible
comments: true
copyright: true
---

## playbook高级特性

### tags 标签

给task打上标签可以允许playbook执行的时候使用--tags选项只执行某个task或者--skip-tags选项不执行某个task.例如:

```
tasks:

    - yum:
        name: "{{ item }}"
        state: installed
      loop:
         - httpd
         - memcached
      tags:
         - packages

    - template:
        src: templates/src.j2
        dest: /etc/foo.conf
      tags:
         - configuration
```

如果只想执行"configuration"和"packags"标签的task,只需要执行:

```
ansible-playbook example.yml --tags "configuration,packages"
```

或者如果想跳过configuration的task.只需要--skip-tags选项

```
ansible-playbook example.yml --skip-tags  "configuration"
```

> tags标签可以复用.可以为多个task打上同一个tags

#### tags继承性

tags可以定义在整个playbook中,或者roles中:

```
- hosts: all
  tags:
    - bar
  tasks:
    ...

- hosts: all
  tags: ['foo']
  tasks:
    ...
```

```
roles:
  - role: webserver
    vars:
      port: 5000
    tags: [ 'web', 'foo' ]
```

---

### Blocks 块功能

Blocks块可以允许将一个或多个task归为一组,例如:

```
 tasks:
   - name: Install Apache
     block:
       - yum:
           name: "{{ item }}"
           state: installed
         with_items:
           - httpd
           - memcached
       - template:
           src: templates/src.j2
           dest: /etc/foo.conf
       - service:
           name: bar
           state: started
           enabled: True
     when: ansible_distribution == 'CentOS'
     become: true
     become_user: root
```

这些task都被放置与一个block中,而且在block中定义了一个whn条件判断语句.这样就不用再每个task中都定义一个相同的when语句.

#### block块处理异常任务.

block可以用来处理任务的异常.有点类似于python编程语句的try exception...finally语句捕获异常.例如:

```
 tasks:
 - name: Attempt and graceful roll back demo
   block:
     - debug:
         msg: 'I execute normally'
     - command: /bin/false
     - debug:
         msg: 'I never execute, due to the above task failing'
   rescue:
     - debug:
         msg: 'I caught an error'
     - command: /bin/false
     - debug:
         msg: 'I also never execute :-('
   always:
     - debug:
         msg: "This always executes"
```

rescue表示上一个task(command:/bin/false)语句执行异常时就会执行rescue的task.

而always表示无论command:/bin/false是否执行异常都会执行.

---

### includes 

includes在Ansible中起引用功能,.其功能非常强大,可以引入一个Playbook文件,变量var文件等等.有时候多个task或者playbook需要进行一项重复的工作,则可以将这部分功能单独写入一个playbook文件,然后再通过includes调用.而不必每个playbook都去写同样一个功能的task.这有点类似于shell脚本的函数

用法:

比如下面定义了一个php的playbook:

```
- hosts: all
  remote_user: root
  tasks:
    - name:PHP a project
      command: A project
- name: restart php
  hosts: all
  remote_user: root
  tasks:
    - include: restartphp.yml
```

然后定义restartphp.yml文件:

```
- name: restart php
  server: name=php-fpm state=restarted
```



同样其他的playbook想要重启Php服务不必再写重复的playbook task.只需要include retartphp.yml文件即可.

#### 动态includes

includes还可以结合when语句.即只有当满足when条件时,才include文件执行.例如:

```
-name: check if file exist
 stat: path=test.txt
 register: check_file

-include: task.yml
 when: check_file.stat.exists
```

下面的一个简单的例子演示了include的用法:

```
[root@localhost playbook]$vim include.yml

---
  - hosts: all
    remote_user: root
    tasks:
       - include: task.yml
         when: ansible_default_ipv4.address == "10.0.4.230"


[root@localhost playbook]$vim task.yml

---
    #这是include.yml文件导入的playbook
  - name: create a file
    file: path=/tmp/include.txt state=touch
```

执行结果可以显示这个include的task.yml文件只在10.0.4.230这台服务器上执行了:

```
[root@localhost playbook]$ansible-playbook include.yml

TASK [create a file] *********************************************************************************************************************************************************************************************
skipping: [10.0.4.231]
changed: [10.0.4.230]

PLAY RECAP *******************************************************************************************************************************************************************************************************
10.0.4.230                 : ok=2    changed=1    unreachable=0    failed=0
10.0.4.231                 : ok=1    changed=0    unreachable=0    failed=0

```

---

### template 模板

template常被用来传输文件,但是template模板的强大之处就在于支持变量替换.template支持jinja2的渲染格式,.另外还支持for循环以及if判断语句.下面来一个简单的例子:

```
#编写一个简单的playbook
[root@localhost playbook]$vim template.yml

---
 - hosts: all
   vars:
      - version: 1.3.5
      - env_name: beta
      - author: jesse

   tasks:
      - name: practise tempalte function
        template: src=template/test_template.j2 dest=/tmp/test_template.txt
        
        
#编写test_template.j2模板文件:
[root@localhost playbook]$vim template/test_template.j2

project_env: {{ env_name }}
project_version: {{ version }}
project_author: {{ author }}

```

执行template.yml这个playbook:

```
[root@localhost playbook]$ansible-playbook template.yml

TASK [practise tempalte function] ********************************************************************************************************************************************************************************
changed: [10.0.4.230]
changed: [10.0.4.231]

PLAY RECAP *******************************************************************************************************************************************************************************************************
10.0.4.230                 : ok=2    changed=1    unreachable=0    failed=0
10.0.4.231                 : ok=2    changed=1    unreachable=0    failed=0
```



查看远程主机上的test_template.txt文件:

```
[root@localhost ~]$cat /tmp/test_template.txt
project_env: beta
project_version: 1.3.5
project_author: jesse
```

---

#### template的jinja2 模板for循环

模板for循环的格式如下:

```
{% for item in item_list %}
  ......
{% endfor %}
```

渲染风格和python的django的模板渲染风格一模一样,. 大括号两边也要预留一个空格..

例如稍微改一下上面例子中的test_template.j2模板文件:

```
[root@localhost playbook]$vim template/test_template.j2

{% for item in range(1,10) %}
   line {{ item }}
{% endfor %}

```

重新执行playbook后,在远程服务器节点上查看/tmp/test_template.txt文件内容:

```
[root@localhost ~]$cat /tmp/test_template.txt
   line 1
   line 2
   line 3
   line 4
   line 5
   line 6
   line 7
   line 8
   line 9
[root@localhost ~]$

```

---

#### template模板的 jinja2 If语句

if条件判断语句格式如下:

```
{% if condition %}
......
{% else%}
{% endif %}
```

继续稍微改一下上面例子的test_template.j2模板文件:

```
#下面的例子中先判断author值,以及myname变量是否定义.(我们的template.yml的playbook文件中定义了author变量,但是没有定义myname变量)
[root@localhost playbook]$vim template/test_template.j2

{% if author == "jesse" %}
HI,my name is jesse
{% endif %}

{% if  myname  is defined %}
hi.myname is {{ myname }}
{% else %}
sorry,myname is not defined.
{% endif %}
```

执行完毕后,远程服务器节点文件内容如下

```
[root@localhost ~]$cat /tmp/test_template.txt
HI,my name is jesse

sorry,myname is not defined.
```

---

#### template jinja2 default()语句

除了if条件判断外,还可以使用default()语句.顾名思义,这是表示一个默认值.

用法格式:

```
{{ var | default(value) }} #如果变量var有定义则取用var的值,否则就使用默认值value
```

修改一下上述的template模板文件如下:

```
[root@localhost playbook]$vim template/test_template.j2

my book is {{ book | default('ansible') }};
my name is {{ author | default('xiaoming') }}
```

执行后,远程服务器节点的文件内容输出如下:

```
[root@localhost ~]$cat /tmp/test_template.txt
my book is ansible;
my name is jesse
```

##### template的Jinja2特性还可以使用python的其他语法,比如合并变量值,过滤等等.这些高级用法以后面对复杂的大项目时再去研究.目前掌握这些基础语法已经组足够满足绝大多数的工作任务

