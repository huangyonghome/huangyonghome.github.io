## ansible--loop循环

#### 标准循环

下面是一个简单的标准loop循环的例子:

```
- name: add several users
  user:
    name: "{{ item }}"
    state: present
    groups: "wheel"
  loop:
     - testuser1
     - testuser2
```

