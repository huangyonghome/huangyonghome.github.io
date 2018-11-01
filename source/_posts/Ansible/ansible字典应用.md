title:  Ansible 字典
date: 2018-11-1 17:59:58
tags:  Ansible
categories: Ansible
comments: true
copyright: true

---

## ansible字典应用

首先定义一个字典变量

例如:

```
vars:

  dict:
    key1: value1
    key2: value2
    key3: value3
    
```

然后在task调用dict:

<!--more-->


    