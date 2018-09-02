## Ansible 条件选择

playbook也可以像shell脚本的if语句那样,基于一个变量的结果来判断是否应该执行某个task.只是ansible的逻辑判断和语法上要别扭,复杂点.

---

### when语句

when语句的条件判断使用非常简单,一般包含2种用法:

1.基于变量值来判断是否应该执行某个task:

```

---
 - hosts: all
   tasks:
     - name: "shutdown Debian flavored systems"
       file: path=/tmp/when state=touch
       when: ansible_default_ipv4.address == "10.0.4.230"
```



执行结果:

```
[root@localhost playbook]$ansible-playbook when.yaml

PLAY [all] *******************************************************************************************************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************************************************************************************
ok: [10.0.4.231]
ok: [10.0.4.230]

TASK [shutdown Debian flavored systems] **************************************************************************************************************************************************************************
skipping: [10.0.4.231]
changed: [10.0.4.230]

PLAY RECAP *******************************************************************************************************************************************************************************************************
10.0.4.230                 : ok=2    changed=1    unreachable=0    failed=0
10.0.4.231                 : ok=1    changed=0    unreachable=0    failed=0

```

可以看到只在230这个IP上执行了动作.

另外,还可以基于or 或者 and的条件判断.例如:

```
tasks:
  - name: "shut down CentOS 6 and Debian 7 systems"
    command: /sbin/shutdown -t now
    when: (ansible_distribution == "CentOS" and ansible_distribution_major_version == "6") or
          (ansible_distribution == "Debian" and ansible_distribution_major_version == "7")
```



或者and,同时满足2个条件

```
tasks:
  - name: "shut down CentOS 6 systems"
    command: /sbin/shutdown -t now
    when:
      - ansible_distribution == "CentOS"
      - ansible_distribution_major_version == "6"
```

> note: 也可以写成 when: ansible_distribution == "CentOS" and ansible_distribution_major_version == "6"



2.基于某个task执行的成功与否作为条件.例如.执行 ls /home/work这个动作,来判断如果有这个文件,则创建个软链.此时就要忽略ls /home/work 这个动作可能出现错误(文件不存在).

```
---
 - hosts: all
   tasks:
     - name: "test file"
       command: ls /home/work
       ignore_errors: True
       register: result

     - name: "create link"
       file: src=/home/work dest=/tmp/work  state=link
       when: result is succeeded

     - name: "create /tmp/home/work"
       file: src=/tmp/home/work state=directory
       when: result is failed
```

register注册一个result的变量,该变量是ls /home/work这个task的执行结果..然后when条件判断当result执行成功,或者执行失败时,才执行相应的task任务

---

### when结合loop循环

变量注册的结果可以是字符串,布尔值,也可以是列表.使用"loop"或者"with_items"关键字可以对变量进行循环.例如:

```
- name: registered variable usage as a loop list
  hosts: all
  tasks:

    - name: retrieve the list of home directories
      command: ls /home
      register: home_dirs

    - name: add home dirs to the backup spooler
      file:
        path: /tmp/{{ item }}
        src: /home/{{ item }}
        state: link
      loop: "{{ home_dirs.stdout_lines }}"
      # same as loop: "{{ home_dirs.stdout.split() }}"
```

执行结果:

```
[root@localhost ~]# ls /home
jesse  tom  tony
[root@localhost ~]# ls /tmp -l
total 0
lrwxrwxrwx 1 root root 11 Aug  3 10:57 jesse -> /home/jesse
lrwxrwxrwx 1 root root  9 Aug  3 10:57 tom -> /home/tom
lrwxrwxrwx 1 root root 10 Aug  3 10:57 tony -> /home/tony

```

