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

