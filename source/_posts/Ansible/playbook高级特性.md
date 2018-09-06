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
- hosts: all
  remote_user: root
  tasks:
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

