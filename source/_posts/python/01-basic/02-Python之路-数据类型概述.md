---

title:  02.Python之路 - python数据类型和变量
date: 2020-05-26 22:59:58
tags:  python
categories: [python,basic]
comments: true
copyright: true

---

### python数据类型和变量



#### 整数

表示数字,包括负数,例如: `1`，`100`，`-8080`，`0`，等等。



#### 浮点数

浮点数也就是小数,可以用数字写法,例如`1.23`，`3.14`，`-9.01`，等等.要注意的是在Python中,除法的结果都是浮点数,即使是被整除,例如:

```
>>> 9/3
3.0
>>> type(9/3)
<class 'float'>
```

在python中,还有一种除法是`//`，称为地板除，两个整数的除法仍然是整数：

```
>>> 10 / 3
3.3333333333333335
>>> 10 // 3
3
>>> type(10 // 3)
<class 'int'>
```

`//`除法只取结果的整数部分，%则表示取余，可以得到两个整数相除的余数：

```
>>> 10 % 3
1
```

----

<!--more-->

#### 字符串

字符串是以单引号`'`或双引号`"`括起来的任意文本，比如`'abc'`，`"xyz"`等等。请注意，`''`或`""`本身只是一种表示方式，不是字符串的一部分，因此，字符串`'abc'`只有`a`，`b`，`c`这3个字符。如果`'`本身也是一个字符，那就可以用`""`括起来，比如`"I'm OK"`包含的字符是`I`，`'`，`m`，空格，`O`，`K`这6个字符。

转义字符`\`可以转义很多字符，比如`\n`表示换行，`\t`表示制表符，字符`\`本身也要转义，所以`\\`表示的字符就是`\`，可以在Python的交互式命令行用`print()`打印字符串看看：

```
>>> print('I\'m ok.')
I'm ok.
>>> print('I\'m learning\nPython.')
I'm learning
Python.
>>> print('\\\n\\')
\
\
```

Python还允许用`r''`表示`''`内部的字符串默认不转义，可以自己试试：

```
>>> print('\\\t\\')
\       \
>>> print(r'\\\t\\')
\\\t\\
```

如果字符串内部有很多换行，用`\n`写在一行里不好阅读，为了简化，Python允许用`'''...'''`的格式表示多行内容，可以自己试试：

```
>>> print('''line1
... line2
... line3''')
line1
line2
line3
```

---

#### 布尔值

布尔值和布尔代数的表示完全一致，一个布尔值只有`True`、`False`两种值，要么是`True`，要么是`False`，在Python中，可以直接用`True`、`False`表示布尔值（请注意大小写），也可以通过布尔运算计算出来：

```
print(3 > 2)
print(3 > 5)

执行结果:
True
False
```

布尔值可以用`and`、`or`和`not`运算。

`and`运算是与运算，只有所有都为`True`，`and`运算结果才是`True`：

```
代码示例:
print(3 > 2 and 3 < 5)
print(3 > 1 and 3 < 1)

执行结果
True
False
```

`or`运算是或运算，只要其中有一个为`True`，`or`运算结果就是`True`：

```
代码示例:

# 布尔值的or运算
print (True or False)
print (3 > 9 or 3 < 5)

执行结果:
True
True
```

`not`运算是非运算，它是一个单目运算符，把`True`变成`False`，`False`变成`True`：

```
#布尔值的Not运算

print(not True)
print(not 3 > 9)

执行结果:
False
True
```

空值不为True，比如：

```
代码示例:

a = ""
b = None

if a:
    print("a is True")
else:
    print("a is False")

if b:
    print("b is True")
else:
    print("b is False")
    
执行结果:

a is False
b is False
```

---

#### 变量

关于变量其实非常简单,用过shell或者其他任何脚本语言,编程语言的都已经非常熟悉了,.但是这里想说一下python中变量和内存地址空间的关系.

例如.当我们给一个变量赋值

```
a = 'ABC
```

时,Python解释器干了两件事情:

1. 在内存中创建了一个`'ABC'`的字符串；
2. 在内存中创建了一个名为`a`的变量，并把它指向`'ABC'`。

也可以把一个变量`a`赋值给另一个变量`b`，这个操作实际上是把变量`b`指向变量`a`所指向的数据，例如下面的代码：

```
a = 'ABC'
b = a
a = 'XYZ'
print(b)
```

最后一行打印出变量`b`的内容到底是`'ABC'`呢还是`'XYZ'`？如果从数学意义上理解，就会错误地得出`b`和`a`相同，也应该是`'XYZ'`，但实际上`b`的值是`'ABC'`，让我们一行一行地执行代码，就可以看到到底发生了什么事：

执行`a = 'ABC'`，解释器创建了字符串`'ABC'`和变量`a`，并把`a`指向`'ABC'`：

![py-var-code-1](https://www.liaoxuefeng.com/files/attachments/923791878255456/0)

执行`b = a`，解释器创建了变量`b`，并把`b`指向`a`指向的字符串`'ABC'`：

![](https://www.liaoxuefeng.com/files/attachments/923792058613440/0)

执行`a = 'XYZ'`，解释器创建了字符串'XYZ'，并把`a`的指向改为`'XYZ'`，但`b`并没有更改：

![](https://www.liaoxuefeng.com/files/attachments/923792191637760/0)

所以，最后打印变量`b`的结果自然是`'ABC'`了。



