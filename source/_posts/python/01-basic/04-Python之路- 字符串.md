---
title:  04.Python之路 - 字符串
date: 2020-05-27 22:59:58
tags:  python
categories: [python,01-basic]
comments: true
copyright: true

---



# Python之路 - 字符串

## 介绍  🍀

字符串是Python中最基本的数据类型之一 

字符串的使用需要用引号括起来 , 例如 : `name = "Lyon"` ; 这里name就是一个变量名 , 而引号里面的`Lyon` 则就是该变量绑定的值 , 该值的类型为 " str" 类型 , 我们可以利用`type()` 函数进行查看 : 

```python
>>> name = "Lyon"
>>> type(name)
<class 'str'>
>>>
```

这就是字符串类型 , 当然如上使用的是双引号 , 这里其实还可以使用单引号`'Lyon'`以及三引号`'''Lyon'''`(或者是`"""Lyon"""`  , 单引号双引号都可以) , 不过对于三引号 , 我们通常是表示多行字符串 , 这样我们就不需要利用 " \n " （换行符）来进行每一行的换行了

对于嵌套引号的时候要注意 , 需要用不同的引号来避免歧义 , 比如 : `'I am "Lyon"'`  , 也可以 `"I am 'Lyon'"` 

对于所有的基本数据类型 , 我们都应该熟悉其特性以及操作

字符串操作主要有 **拷贝（复制）、拼接、查找、统计、切片、测试、大小写,字符串列表转换等**

在开始详细了解这些操作之前 , 我们需要记住一个特性 : **字符串是不可变的** , 既然是不可变的 , 那么我们对其进行的增删改查就都不是对本身进行操作的 , 而是创建了一个新的字符串

<!--more-->

## 拷贝  🍀

```python
>>> a = "Lyon"
>>> b = a
>>> print(a,b)
Lyon Lyon
```

## 拼接  🍀

```python
>>> a = "Hello"
>>> b = "Lyon"
>>> print(a+b)
HelloLyon
```

注 : 这个方法要特别说明一下 , “+”是一个坑 , 因为使用加号连接2个字符串会调用静态函数`string_concat(register PyStringObject *a,register PyObject *b)`  , 这个函数大致的作用 , 就是首先开辟一块`a+b`大小的内存的和的存储单元 , 然后把a和b都拷贝进去 ; 所以一旦我们的 "+" 操作过多将会造成大量内存的浪费

```python
>>> a = "Lyon"
>>> b = "Hello"
>>> print(a.join(b)) 
HLyoneLyonlLyonlLyono  #HLyon eLyon lLyon lLyon o
```

可以用join来将list中的元素进行拼接成字符串 : `''.join( list )` 即以空字符串连接列表中的每一个元素

下面2个例子演示了.默认用空格,以及用逗号来拼接字符串的效果:

```
b = 'jesse'
print("".join(b)) #用空格拼接,字符串不变
print(",".join(b)) #将字符串b的每个元素用逗号进行拼接

执行结果
jesse
j,e,s,s,e
```
字符串拼接还有个方法是用格式化输出.下面是这3个方法的具体案例.可以看到格式化输出是最简单的拼接方法

```
#join方法要求先生成一个列表,而且要将整数型转换成字符串

a = 'world'
b = 1
while b < 10:
    l1 = [a,str(b)]
    print(''.join(l1))
    b+=1



#格式化输出可以直接打印拼接字符串
a = 'hello'
b = 1

while b < 10:
    print("%s%d" %(a,b))
    b+=1


#+方法需要2个变量都是字符串.要将整数型转换成字符串
a = 'hello'
b = 1

while b < 5:
    print(a+str(b))
    b+=1
```


## 查找  🍀

* index()

```python
>>> name = "Lyon"
# 返回L字符所在的下标,下标是从0开始的整数
>>> name.index('L')
0 
# 如果不存在就会报错
>>> name.index('N') 
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ValueError: substring not found  
# 也可以用in,not in来进行判断
>>>'L' in name
>>>
```

* find()

find的函数和index类似,都是返回字符的下标.而且都是如果字符串中有多个相同的元素,则返回第一个匹配的元素下标

```
b = 'jesse'
print(b.find('e))

>>> 1
#和index区别是,find如果没有找到匹配的字符,会返回-1
print(b.find('a'))
>>> -1

#当然,两者都支持定义起始位置,即从某个起始位置开始查找.

print(b.find('e',2)) #从第二个位置开始查找
>>> 4
print(b.find('e',2,3)) #从第二个位置开始查找,但是只查找到第三个结束,所以没有找到e字符串
>>> -1

print(b.index('e',2)) #从第二个位置开始查找
>>> 4

print(b.index('e',2,3)) 
Traceback (most recent call last):
4
  File "/Users/huangyong/Desktop/python-learning/day1/python字符串.py", line 43, in <module>
    print(b.index('e',2,3))
ValueError: substring not found
```

---

## 统计  🍀

```python
#count表示统计字符串中某个元素的个数
b = 'jesse'
print(b.count('e')) 
>>> 2
#len表示字符串的长度
print(len(b))
>>> 5
```

---

## 切片  🍀

```python
#切片就是通过索引（索引：索引：步长）截取字符串的一段，形成新的字符串（原则就是顾头不顾腚).

b = 'jesse'
print(b[0:3]) #从第0个下标开始截取到第3个..如果起始位置为0.则0可以不用写
>>> jes
#等同于
print(b[:3])

>>> name = "i like Lyon"
# 切取第7个到第9个字符,注意空格也是一个字符
>>> name[7:10]     
'Lyo'
>>> name = "i like Lyon"
# 第7到第10各,顾头不顾尾
>>> name[7:11]
'Lyon'

#如果是全部截取,则不需要加起始值和截止值
print(b[:])
>>> jesse

#如果是反向,从最末尾开始倒着截取,则用负数下标
print(b[-1:]) #输出最后一个字符
>>> e
print(b[-1:-3:-1]) #输出最后一个字符到倒数第三个字符,注意这里要用-1表示步长,因为是倒序匹配,所以步长也是倒序
>>> es
print(b[::-1]) #倒序打印,也就是字符串翻转
>>> essej

#步长表示每隔多少个间隔截取字符串
print(b[::-2])
>>> esj

```

## 检测  🍀

```python
>>> name = "Lyon"
# 检测"L"是否在name中,返回bool值
>>> "L" in name     
True
>>> num = "3412313"
# 检测num里面是否全都是整数
>>> num.isdigit()    
True
>>> name = "Lyon"
# 检测name是否可以被当作标标志符,即是否符合变量命名规则 
>>> name.isidentifier()
True　
```

检测相关

```python
#两个判断字符串是否以某字符开头(结尾)的方法
str.startswith(prefix[,start[,end]]) # 是否以prefix开头 
str.endswith(suffix[,start[,end]])   # 以suffix结尾 

print(b.startswith('j'))
>>> True
print(b.endswith('e'))
>>> True

# 还有其他一些检测相关的判断函数
str.isalnum()    # 是否全是字母和数字,并至少有一个字符 
str.isalpha()    # 是否全是字母,并至少有一个字符 
str.isdigit()    # 是否全是数字,并至少有一个字符 
str.isspace()    # 是否全是空白字符,并至少有一个字符 
str.islower()    # 是否全是小写 
str.isupper()    # 是否便是大写 
str.istitle()    # 是否是首字母大写的
```

注 : 结果全是bool值

## 大小写  🍀

```python
>>> name = "I am Lyon"
# 大小写互换
>>> name.swapcase()   
'i AM lYON'
# 首字母大写,其它都小写
>>> name.capitalize()     
'I am lyon'
# 转换为大写
>>> name.upper()          
'I AM LYON'
# 转换为小写
>>> name.lower()           
'i am lyon'
```

## split()  🍀

split表示将一串字符串以列表形式返回.默认分隔符为空格.注意最终返回的列表中,不包含分隔字符

```
b = 'jeese hello world'
print(b.split())
>>> ['jeese', 'hello', 'world']

b = '/usr/bin:/usr/sbin:/usr/local/bin:/bin:/sbin'
print(b.split(':'))
>>> ['/usr/bin', '/usr/sbin', '/usr/local/bin', '/bin', '/sbin']
```

## strip()  🍀

```
#strip表示截去首尾的所有空格.
b= '    jesse     '
print(b.strip())
>>> jesse
#lstrip表示截去首部所有空格
print(b.lstrip())
#rstrip表示截去尾部所有空格
print(b.rstrip())
```

---

## replace()  🍀

```
#指定要替换的字符串,用指定的新字符串替换
b = 'jesse'
print(b.replace('s','a'))
>>> jeaae
```

---

## 更多  🍀

```python
 |  capitalize(...)
 |      S.capitalize() -> str
 |
 |      Return a capitalized version of S, i.e. make the first character
 |      have upper case and the rest lower case.
 |
 |  casefold(...)
 |      S.casefold() -> str
 |
 |      Return a version of S suitable for caseless comparisons.
 |
 |  center(...)
 |      S.center(width[, fillchar]) -> str
 |
 |      Return S centered in a string of length width. Padding is
 |      done using the specified fill character (default is a space)
 |
 |  count(...)
 |      S.count(sub[, start[, end]]) -> int
 |
 |      Return the number of non-overlapping occurrences of substring sub in
 |      string S[start:end].  Optional arguments start and end are
 |      interpreted as in slice notation.
 |
 |  encode(...)
 |      S.encode(encoding='utf-8', errors='strict') -> bytes
 |
 |      Encode S using the codec registered for encoding. Default encoding
 |      is 'utf-8'. errors may be given to set a different error
 |      handling scheme. Default is 'strict' meaning that encoding errors raise
 |      a UnicodeEncodeError. Other possible values are 'ignore', 'replace' and
 |      'xmlcharrefreplace' as well as any other name registered with
 |      codecs.register_error that can handle UnicodeEncodeErrors.
 |
 |  endswith(...)
 |      S.endswith(suffix[, start[, end]]) -> bool
 |
 |      Return True if S ends with the specified suffix, False otherwise.
 |      With optional start, test S beginning at that position.
 |      With optional end, stop comparing S at that position.
 |      suffix can also be a tuple of strings to try.
 |
 |  expandtabs(...)
 |      S.expandtabs(tabsize=8) -> str
 |
 |      Return a copy of S where all tab characters are expanded using spaces.
 |      If tabsize is not given, a tab size of 8 characters is assumed.
 |
 |  find(...)
 |      S.find(sub[, start[, end]]) -> int
 |
 |      Return the lowest index in S where substring sub is found,
 |      such that sub is contained within S[start:end].  Optional
 |      arguments start and end are interpreted as in slice notation.
 |
 |      Return -1 on failure.
 |
 |  format(...)
 |      S.format(*args, **kwargs) -> str
 |
 |      Return a formatted version of S, using substitutions from args and kwargs.
 |      The substitutions are identified by braces ('{' and '}').
 |
 |  format_map(...)
 |      S.format_map(mapping) -> str
 |
 |      Return a formatted version of S, using substitutions from mapping.
 |      The substitutions are identified by braces ('{' and '}').
 |
 |  index(...)
 |      S.index(sub[, start[, end]]) -> int
 |
 |      Like S.find() but raise ValueError when the substring is not found.
 |
 |  isalnum(...)
 |      S.isalnum() -> bool
 |
 |      Return True if all characters in S are alphanumeric
 |      and there is at least one character in S, False otherwise.
 |
 |  isalpha(...)
 |      S.isalpha() -> bool
 |
 |      Return True if all characters in S are alphabetic
 |      and there is at least one character in S, False otherwise.
 |
 |  isdecimal(...)
 |      S.isdecimal() -> bool
 |
 |      Return True if there are only decimal characters in S,
 |      False otherwise.
 |
 |  isdigit(...)
 |      S.isdigit() -> bool
 |
 |      Return True if all characters in S are digits
 |      and there is at least one character in S, False otherwise.
 |
 |  isidentifier(...)
 |      S.isidentifier() -> bool
 |
 |      Return True if S is a valid identifier according
 |      to the language definition.
 |
 |      Use keyword.iskeyword() to test for reserved identifiers
 |      such as "def" and "class".
 |
 |  islower(...)
 |      S.islower() -> bool
 |
 |      Return True if all cased characters in S are lowercase and there is
 |      at least one cased character in S, False otherwise.
 |
 |  isnumeric(...)
 |      S.isnumeric() -> bool
 |
 |      Return True if there are only numeric characters in S,
 |      False otherwise.
 |
 |  isprintable(...)
 |      S.isprintable() -> bool
 |
 |      Return True if all characters in S are considered
 |      printable in repr() or S is empty, False otherwise.
 |
 |  isspace(...)
 |      S.isspace() -> bool
 |
 |      Return True if all characters in S are whitespace
 |      and there is at least one character in S, False otherwise.
 |
 |  istitle(...)
 |      S.istitle() -> bool
 |
 |      Return True if S is a titlecased string and there is at least one
 |      character in S, i.e. upper- and titlecase characters may only
 |      follow uncased characters and lowercase characters only cased ones.
 |      Return False otherwise.
 |
 |  isupper(...)
 |      S.isupper() -> bool
 |
 |      Return True if all cased characters in S are uppercase and there is
 |      at least one cased character in S, False otherwise.
 |
 |  join(...)
 |      S.join(iterable) -> str
 |
 |      Return a string which is the concatenation of the strings in the
 |      iterable.  The separator between elements is S.
 |
 |  ljust(...)
 |      S.ljust(width[, fillchar]) -> str
 |
 |      Return S left-justified in a Unicode string of length width. Padding is
 |      done using the specified fill character (default is a space).
 |
 |  lower(...)
 |      S.lower() -> str
 |
 |      Return a copy of the string S converted to lowercase.
 |
 |  lstrip(...)
 |      S.lstrip([chars]) -> str
 |
 |      Return a copy of the string S with leading whitespace removed.
 |      If chars is given and not None, remove characters in chars instead.
 |
 |  partition(...)
 |      S.partition(sep) -> (head, sep, tail)
 |
 |      Search for the separator sep in S, and return the part before it,
 |      the separator itself, and the part after it.  If the separator is not
 |      found, return S and two empty strings.
 |
 |  replace(...)
 |      S.replace(old, new[, count]) -> str
 |
 |      Return a copy of S with all occurrences of substring
 |      old replaced by new.  If the optional argument count is
 |      given, only the first count occurrences are replaced.
 |
 |  rfind(...)
 |      S.rfind(sub[, start[, end]]) -> int
 |
 |      Return the highest index in S where substring sub is found,
 |      such that sub is contained within S[start:end].  Optional
 |      arguments start and end are interpreted as in slice notation.
 |
 |      Return -1 on failure.
 |
 |  rindex(...)
 |      S.rindex(sub[, start[, end]]) -> int
 |
 |      Like S.rfind() but raise ValueError when the substring is not found.
 |
 |  rjust(...)
 |      S.rjust(width[, fillchar]) -> str
 |
 |      Return S right-justified in a string of length width. Padding is
 |      done using the specified fill character (default is a space).
 |
 |  rpartition(...)
 |      S.rpartition(sep) -> (head, sep, tail)
 |
 |      Search for the separator sep in S, starting at the end of S, and return
 |      the part before it, the separator itself, and the part after it.  If the
 |      separator is not found, return two empty strings and S.
 |
 |  rsplit(...)
 |      S.rsplit(sep=None, maxsplit=-1) -> list of strings
 |
 |      Return a list of the words in S, using sep as the
 |      delimiter string, starting at the end of the string and
 |      working to the front.  If maxsplit is given, at most maxsplit
 |      splits are done. If sep is not specified, any whitespace string
 |      is a separator.
 |
 |  rstrip(...)
 |      S.rstrip([chars]) -> str
 |
 |      Return a copy of the string S with trailing whitespace removed.
 |      If chars is given and not None, remove characters in chars instead.
 |
 |  split(...)
 |      S.split(sep=None, maxsplit=-1) -> list of strings
 |
 |      Return a list of the words in S, using sep as the
 |      delimiter string.  If maxsplit is given, at most maxsplit
 |      splits are done. If sep is not specified or is None, any
 |      whitespace string is a separator and empty strings are
 |      removed from the result.
 |
 |  splitlines(...)
 |      S.splitlines([keepends]) -> list of strings
 |
 |      Return a list of the lines in S, breaking at line boundaries.
 |      Line breaks are not included in the resulting list unless keepends
 |      is given and true.
 |
 |  startswith(...)
 |      S.startswith(prefix[, start[, end]]) -> bool
 |
 |      Return True if S starts with the specified prefix, False otherwise.
 |      With optional start, test S beginning at that position.
 |      With optional end, stop comparing S at that position.
 |      prefix can also be a tuple of strings to try.
 |
 |  strip(...)
 |      S.strip([chars]) -> str
 |
 |      Return a copy of the string S with leading and trailing
 |      whitespace removed.
 |      If chars is given and not None, remove characters in chars instead.
 |
 |  swapcase(...)
 |      S.swapcase() -> str
 |
 |      Return a copy of S with uppercase characters converted to lowercase
 |      and vice versa.
 |
 |  title(...)
 |      S.title() -> str
 |
 |      Return a titlecased version of S, i.e. words start with title case
 |      characters, all remaining cased characters have lower case.
 |
 |  translate(...)
 |      S.translate(table) -> str
 |
 |      Return a copy of the string S in which each character has been mapped
 |      through the given translation table. The table must implement
 |      lookup/indexing via __getitem__, for instance a dictionary or list,
 |      mapping Unicode ordinals to Unicode ordinals, strings, or None. If
 |      this operation raises LookupError, the character is left untouched.
 |      Characters mapped to None are deleted.
 |
 |  upper(...)
 |      S.upper() -> str
 |
 |      Return a copy of S converted to uppercase.
 |
 |  zfill(...)
 |      S.zfill(width) -> str
 |
 |      Pad a numeric string S with zeros on the left, to fill a field
 |      of the specified width. The string S is never truncated.
 |
 |  ----------------------------------------------------------------------
```
