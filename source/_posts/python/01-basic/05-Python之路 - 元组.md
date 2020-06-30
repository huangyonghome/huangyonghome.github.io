---
title:  05.Python之路 - 元组
date: 2020-05-27 22:59:58
tags:  python
categories: [python,01-basic]
comments: true
copyright: true

---

# Python之路 - 元组

## 介绍  🍀

元组和字符串一样 , 元组也是不可修改的 , 元组使用小括号

元组的创建很简单 , 只需要在括号中添加元素 , 并使用逗号隔开即可

元组需要掌握的分别有**创建、访问、修改、切片、删除、检测等**

## 创建  🍀

```python
# 创建一个带有元素的元组
mytuple = ("Lyon", "Alex", "Leon", 1, 2, 3)
# 也可以不加括号,但是一定要加引号
mytuple = "Lyon", "Alex", "Leon", 1, 2, 3
# 创建一个空元组
mytuple = ()
# 当元组中只有一个元素,加逗号来消除歧义哟,这是一个好习惯,因为()既可以表示tuple又可以表示数学公式中的小括号
only_one = ("Lyon",)
```

<!--more-->

## 访问  🍀

```python
# 创建一个元组
names = ("Lyon", "Alex", "Leon", 1, 2, 3)
# 访问元组中的第一个元素并打印结果,下标索也是从0开始
print(names[0]) 
# 访问元组中第一和第二个元素并打印结果
print(names[0:2])
'''
打印结果:
Lyon
('Lyon', 'Alex')
'''
```

注意 : 结果发现访问单个元素结果是字符串类型 , 而多个则是元组类型

字符串中每个字符就是一个元素 , 而元组中则是一个数字或者一个字符串就是一个元素

## 修改  🍀

```python
# 创建一个元组
tuple_name = ("Lyon", "Alex", "Leon", 1, 2, 3)
# 创建另一个元组
tuple_num = (1, 2, 3, 4, 5)
# 生成一个新的元组
tuple_total = tuple_name + tuple_num
# 打印tuple_total
print(tuple_total) 
# 复制元组内元素一次
tuple_total = tuple_name * 2
# 打印tuple_total看结果
print(tuple_total)     
# 在列表中可以通过索引取值后进行修改,但是元组里面是非法的哦
tuple_name[0] = "lyon"      # 这里直接就报错
'''
打印结果:
('Lyon', 'Alex', 'Leon', 1, 2, 3, 1, 2, 3, 4, 5)
('Lyon', 'Alex', 'Leon', 1, 2, 3, 'Lyon', 'Alex', 'Leon', 1, 2, 3)
'''
```

注意 : 元组是不可变的 , 所以对于所有的修改操作 , 都是在根据原元组生成了一个新的元组

## 删除  🍀

```python
#创建一个元组
names = ("Lyon", "Alex", "Leon", 1, 2, 3)
#元祖可以被整个删除,但是不能删除元祖的元素
del names
del names[0] 
# TypeError: 'tuple' object doesn't support item deletion

```

## 切片  🍀

```python
names = ("Lyon", "Kenneth", "Leon", "Charlie")
# 打印子集,第二个至第三个
print(names[1:2])
# 打印子集,倒数第三个(即第二个)至第三个
print(names[-3:3])
# 打印子集,第一个至第三个,隔一个取一个
print(names[0:2:1])
'''
打印结果:
('Kenneth', 'Leon')
('Kenneth', 'Leon')
('Leon',)
'''
```

## 检测  🍀

```python
# 创建一个元组
tuple_name = ("Lyon","Alex","Leon",1,2,3)
# "Lyon"是否在tuple_name中，打印结果
print("Lyon" in tuple_name)
# 打印结果:True
```

## 更多  🍀

实例

```python
# 创建一个元组
tuple_name = ("Lyon","Alex","Leon",1,2,3)
# 计算元组长度
tuple_len = len(tuple_name)     
# 打印结果
print(tuple_len)     
# 创建一个元素全为数字的元组
tuple_num = (1,2,3,4,5)
# 返回元组中的最大值
print(max(tuple_num))      
# 返回元组中的最小值
print(min(tuple_num))     
# 创建一个列表
list_name = ["Lyon","Alex","Leon"]
# 将列表转换为元组
tuple_names = tuple(list_name)
# 打印tuple_names
print(tuple_names)     
'''
打印结果:
6
5
1
('Lyon', 'Alex', 'Leon')
'''
```

方法

```python
 |  count(...)
 |      T.count(value) -> integer -- return number of occurrences of value
 |
 |  index(...)
 |      T.index(value, [start, [stop]]) -> integer -- return first index of value.
 |      Raises ValueError if the value is not present.
```