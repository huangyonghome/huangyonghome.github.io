---
title:  17.Python之路 - Python递归函数
date: 2020-06-26 09:20:58
tags:  python
categories: [python,03-Modules]
comments: true
copyright: true
---



# Python之路 -  模块初识

##  介绍 🍀

Python 模块 , 说白了就是一个 .py 文件 , 里面放了一坨函数和变量或者类 , 总而言之就是放了一堆代码 , 那么问题来了 , 我要它有什么用?

当我们写一个比较复杂的程序 , 程序里面定义了100个函数和200个变量 , 然后这些函数和变量要来回调用 , 于是我们就得到处找函数名 , 找变量名 ,找了一个小时终于找到了 , 好的 , 我们开始找下一个 ... 

所以 , 这个时候我们就需要模块了 , 我们可以将一类作用的函数和变量放到一个 .py 文件中 , 这样分成好几个文件 , 我们就可以快速维护我们的代码了

<!--more-->

模块的作用:

1. 便于维护

模块导入方式:

```python
#  导入整个模块或者包
1. import moudle
#  导入模块或者包中的所有内容
2. from moudle import *
#  从模块或者包导入一部分内容
3. from moudle import part 
#  从模块或者包导入一部分内容并命名为m
4. from moudle import part as m
```

## collections模块  🍀

在内置数据类型(dict、list、set、tuple)的基础上 , conllections模块还提供了几个额外的数据类型 : Counter 、deque、defaultdict、namedtuple和OrderedDict等

### namedtuple  🍀

生成可以使用名字来访问元素内容的tuple

从名字可以理解 , 带名字的元组 , 即我们可以通过key来取值

官方文档中解释 : 返回一个具有命名字段的元组的新子类

```python
# 从collections模块中导入namedtuple
>>> from collections import namedtuple
# 设置属性
>>> Point = namedtuple('Point',['x','y'])
# 传入参数
>>> p = Point(1,2)
# 利用x进行取值
>>> p.x
1
# 利用y进行取值
>>> p.y
2
```

类似的 , 如果我们定义一个圆

```python
>>> Circle = namedtuple('Circle',['x','y','r'])
```

### deque  🍀

双向队列 , 可以快速的从另外一侧追加和推出对象

使用list存储数据时 , 按索引访问元素很快 , 但是插入和删除元素就很慢了 , 因为list是线性存储 , 数据量大的时候 , 插入和删除效率很低

deque就是为了高校实现插入和删除操作的双向列表 , 适合用于队列和栈:

```python
# 从collections模块中导入deque
>>> from collections import deque
# 创建双向队列
>>> q = deque(['a','b','c'])
# 从最后插入
>>> q.append('x')
# 从头插入
>>> q.appendleft('y')
>>> q
deque(['y','a','b','c','x'])
```

deque除了实现list的 append() 和 pop() 外, 还支持 appendleft() 和 popleft() , 这样就可以非常高效地往头部添加或删除元素

 ###  Counter  🍀

Counter类的目的是用来跟踪值出现的次数 , 它是一个无序的容器类型 , 以字典的键值对形式存储 , 其中元素作为key , 其计数作为value ; 计数值可以是任意interger(包括0和负数) , Counter类和其他语言的bags或multisets很相似

```python
>>> c = Counter('abcdeabcdabcaba')
>>> c
Counter({'a': 5, 'b': 4, 'c': 3, 'd': 2, 'e': 1})
```

 [Counter详细用法](http://www.cnblogs.com/Eva-J/articles/7291842.html)

### OrderedDict  🍀

创建有序字典

使用dict时 , key是无序的 ; 在对dict做迭代时 , 我们无法确定key的顺序 , 这时我们就可以利用OrderedDict来实现我们的目标

```python
# 从collections模块中盗图OrderedDict
>>> from collections import OrderedDict
# 创建一个无序字典
>>> d = dict([('a',1),('b',2),('c',3)])
>>> d # dict的key是无序的
{'a':1,'c':3,'b':2}
# 创建一个有序字典
>>> od = OrderedDict([('a',1),('b',2),('c',3)])
>>> od  # OrderedDict的key是有序的
OrderedDict([('a',1),('b',2),('c',3)])
```

PS : OrderedDict的key会按照插入的顺序排序 , 不是key本身排序:

```python
>>> od = OrderedDict()
# 插入一个键值对
>>> od['z'] = 1
>>> od['y'] = 2
>>> od['x'] = 3
>>> od.keys()  # 按照插入的key的顺序返回
['z','y','x']
```

### defaultdict  🍀

带有默认值的字典

当我们使用dic时 , 如果引用的key不存在 , 就会抛出KeyError ; 如果希望key不存在时 , 返回一个默认值 , 就可以使用defaultdict

```python
# 从collections模块中导入defaultdict
>>> from collections import defaultdict
# 创建带默认值的字典
>>> dd = defaultdict(lambda:None)
# 为创建的字典中添加键值对
>>> dd['key1'] = 'abc'
abc
# key2不存在,自动添加返回默认值
>>> dd['key2']
None
# 查看最后结果
>>> dict(dd)
{'key1': 'abc', 'key2': None}
```

PS : 在dict类中的方法有一个 setdefault() 是用于创建的 ,而defaultdict是用来访问的