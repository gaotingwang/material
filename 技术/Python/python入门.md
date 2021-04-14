[TOC]

一、数据类型
---

1. 整数 

   十六进制用`0x`前缀和0-9，a-f表示，例如：`0xff00`，`0xa5b4c3d2`，等等

2. 浮点数

   对于很大或很小的浮点数，就必须用科学计数法表示，把10用e替代，1.23x10^9就是**1.23e9**，或者**12.3e8**，0.000012可以写成**1.2e-5**，等等

   整数和浮点数混合运算的结果就变成浮点数 `1 + 2.0    # ==> 浮点数 3.0`

3. 字符串

   字符串是以`''`或`""`括起来的任意文本

4. 布尔值

   布尔值只有`True`、`False`两种值，布尔值可以用`and`、`or`和`not`运算。Python解释器在做布尔运算时，只要能提前确定计算结果，它就不会往后算了，**直接返回结果**。

   Python把`0`、`空字符串''`和`None`看成 False，其他数值和非空字符串都看成 True:

   ```python
   print True and 'a=T' # a=T
   ```

5. 空值

   空值是Python里一个特殊的值，用`None`表示。

**Python之print语句**

print语句也可以跟上多个字符串，用逗号“,”隔开，遇到逗号“,”时会输出一个空格,就可以连成一串输出：

```python
>>> print 'The quick brown fox', 'jumps over', 'the lazy dog'
The quick brown fox jumps over the lazy dog
```

print也可以打印整数，或者计算结果：

```python
>>> print '100 + 200 =', 100 + 200
100 + 200 = 300     #运行结果
```

**Python中定义变量**

在Python中，等号`=`是赋值语句，可以把任意数据类型赋值给变量，同一个变量可以反复赋值，而且可以是不同类型的变量，例如：

```python
a = 123    # a是整数
print a
a = 'imooc'   # a变为字符串
print a
```

这种变量本身类型不固定的语言称之为动态语言，与之对应的是静态语言。Java是静态语言，需要指定变量类型。

**Python中raw字符串**

如果一个字符串包含很多需要转义的字符，对每一个字符都进行转义会很麻烦。为了避免这种情况，我们可以在字符串前面加个前缀` r`，表示这是一个 raw 字符串，里面的字符就不需要转义了。`r`的作用就是禁止转义，但对`'`和 `"`无效。

```python
>>> print r'Line 1\nLine 2\nLine 3'
Line 1\nLine 2\nLine 3

# 对应'无法处理
>>> print r'l'' 
File "index.py", line 1
    print r'l''
              ^
SyntaxError: EOL while scanning string literal
```

但是`r'...'`表示法不能表示多行字符串，也不能表示包含`'`和 `"`的字符串。

**Python中多行字符串**

如果要表示多行字符串，可以用`'''...'''`表示：

```python
# 被转义了
>>> print '''Line 1\\
Line 2\\
Line 3'''
Line 1\
Line 2\
Line 3

# 使用raw字符串未被转义
>>> print r'''Line 1\\
Line 2\\
Line 3'''
Line 1\\
Line 2\\
Line 3
```

二、函数编写
---

在Python中，定义一个函数要使用`def `语句，依次写出`函数名`、`括号`、`括号中的参数`和`冒号:`，然后，**在缩进块中编写函数体**，函数的返回值用 `return`语句返回。如果没有return语句，函数执行完毕后也会返回结果，只是结果为 None。*return None可以简写为return。*

```python
def my_abs(x):
    if x >= 0:
        return x
    else:
        return -x
```

在python中函数可以返回多个值：

```python
import math
def move(x, y, step, angle):
    nx = x + step * math.cos(angle)
    ny = y - step * math.sin(angle)
    return nx, ny
```

这样就可以同时获得返回值：

```python
>>> x, y = move(100, 100, 60, math.pi / 6)
>>> print x, y
151.961524227 70.0
```

但其实这只是一种假象，Python函数返回的仍然是单一值：

```python
>>> r = move(100, 100, 60, math.pi / 6)
>>> print r
(151.96152422706632, 70.0)
```

用print打印返回结果，原来返回值是一个**tuple**！

但是，在语法上，返回一个tuple可以省略括号，而多个变量可以同时接收一个tuple，按位置赋给对应的值，所以，**Python的函数**返回多值其实就是**返回一个tuple**，但写起来更方便。

### 默认参数

**函数的默认参数的作用是简化调用**，只需要把必须的参数传进去。但是在需要的时候，又可以传入额外的参数来覆盖默认参数值。

定义一个计算 x 的N次方的函数:

```python
def power(x, n):
    s = 1
    while n > 0:
        n = n - 1
        s = s * x
    return s
```

假设计算平方的次数最多，就可以把 n 的默认值设定为 2：

```python
def power(x, n=2):
    s = 1
    while n > 0:
        n = n - 1
        s = s * x
    return s
```

这样一来，计算平方就不需要传入两个参数了：

```python
>>> power(5)
25
```

由于函数的参数按从左到右的顺序匹配，所以==**默认参数只能定义在必需参数的后面：**==

```python
# OK:
def fn1(a, b=1, c=2):
    pass
# Error:
def fn2(a=1, b):
    pass
```

### 定义可变参数

如果想让一个函数能接受任意个参数，我们就可以定义一个可变参数：

```python
def fn(*args):
    print args
    
>>> fn('a', 'b', 'c')
('a', 'b', 'c')
```

==可变参数的名字前面有个*号==，可以传入0个、1个或多个参数给可变参数

三、模块安装
---

要想模块安装，一般会使用pip3,这是一个安装模块非常有利的工具，例如进行numpy模块安装：

```shell
sudo pip3 install numpy
sudo pip3 install -U numpy 升级模块
```

python中import模块：

```python
# 方式1：
import time

time.localtime()

# 方式2：
import time as t

t.localtime()

# 方式3：
from time import localtime  引入模块的部分功能
from time import *          引入模块的所有功能

localtime()
```

要想引入自己模块，得确保自己的模块与引入模块在同一目录下。

python下载好的模块在python目录下的site-packages/中，自己模块放入此目录中，也可以像其他模块一样引入。

## 四、条件判断和循环

**注意: **Python代码的缩进规则。具有相同缩进的代码被视为代码块。

缩进请严格按照Python的习惯写法：**4个空格，不要使用Tab，更不要混合Tab和空格**，否则很容易造成因为缩进引起的语法错误。

### if判断

 if 语句后接表达式，然后用`:`表示代码块开始。

```python
age = 20
if age >= 18:
    print 'your age is', age
    print 'adult'
print 'END'
```

### if-else

**注意:** else 后面也有个“:”。

```python
if age >= 18:
    print 'adult'
else:
    print 'teenager'
```

### if-elif-else

```python
score = 85
if score >= 90:
    print 'excellent'
elif score >= 80:
    print 'good'
elif score >= 60:
    print 'passed'
else:
    print 'failed'
```

### for循环

for...in

```python
L = ['Adam', 'Lisa', 'Bart']
for name in L:
    print name
```

### while循环

```python
sum = 0
x = 1
while True:
    sum = sum + x
    x = x + 1
    if x > 100:
        break
print sum
```

## 五、List和Tuple类型

- `list`是一种有序的集合，可以随时添加和删除其中的元素。直接用` [ ] `把list的所有元素都括起来，就是一个list对象。由于Python是动态语言，所以list中包含的元素并不要求都必须是同一种数据类型，**完全可以在list中包含各种数据**：
   ```python
   L = ['Michael', 100, True]
   ```
   1. 元素访问，正序L[0]，也可以倒序L[-1]
   2. 添加新元素，第一个办法是用 list 的` append() `方法，追加到 list 的末尾；第二个方法是用list的` insert()`方法，它接受两个参数，第一个参数是索引号，第二个参数是待添加的新元素
   3. 删除元素，`pop()`方法总是删掉list的最后一个元素，` pop(i)`删除指定位置元素
   4. 替换元素， L[-1] = 'Paul'，修改指定位置元素


- tuple是另一种有序的列表，中文翻译为“ 元组 ”。tuple 和 list 非常类似，但是，**tuple一旦创建完毕，就不能修改了**

  获取 tuple 元素的方式和 list 是一模一样的，可以正常使用 t[0]

  创建单元素：因为用()定义单元素的tuple有歧义，所以 Python 规定，单元素 tuple 要多加一个逗号“,”，这样就避免了歧义：

  ```python
  >>> t = (1,)
  >>> print t
  (1,)  # Python在打印单元素tuple时，也自动添加了一个“,”，为了更明确地告诉你这是一个tuple。
  ```

- "可变"tuple

  ```
  >>> t = ('a', 'b', ['A', 'B'])
  ```

  注意到 t 有 3 个元素：**'a'，'b'**和一个list：**['A', 'B']**。list作为一个整体是tuple的第3个元素。list对象可以通过 t[2] 拿到，可以进行修改。

python中**迭代永远是取出元素本身，而非元素的索引。**

对于有序集合，元素确实是有索引的。有的时候，我们确实想在 for 循环中拿到索引，怎么办？

方法是使用 **enumerate() 函数**：

```python
>>> L = ['Adam', 'Lisa', 'Bart', 'Paul']
>>> for index, name in enumerate(L):
...     print index, '-', name
... 
0 - Adam
1 - Lisa
2 - Bart
3 - Paul
```

使用 enumerate() 函数，我们可以在for循环中同时绑定索引index和元素name。但是，这不是 enumerate() 的特殊语法。实际上，enumerate() 函数把：

```python
['Adam', 'Lisa', 'Bart', 'Paul']
```

变成了类似：

```python
[(0, 'Adam'), (1, 'Lisa'), (2, 'Bart'), (3, 'Paul')]
```

因此，迭代的每一个元素实际上是一个tuple：

```python
for t in enumerate(L):
    index = t[0]
    name = t[1]
    print index, '-', name
```

如果我们知道每个tuple元素都包含两个元素，for循环又可以进一步简写为：

```python
for index, name in enumerate(L):
    print index, '-', name
```

这样不但代码更简单，而且还少了两条赋值语句。

可见，索引迭代也不是真的按索引访问，而是由 enumerate() 函数自动把每个元素变成 (index, element) 这样的tuple，再迭代，就同时获得了索引和元素本身。

## 六、Dict和Set类型

- dict相当于map，也可以理解为json串对象，花括号 {} 表示这是一个dict，然后按照 **key: value**, 写出来即可

  **dict的第一个特点是查找速度快，无论dict有10个元素还是10万个元素，查找速度都一样**。而list的查找速度随着元素增加而逐渐下降。

  **dict的第二个特点就是存储的key-value序对是没有顺序的！**这和list不一样

  **dict的第三个特点是作为 key 的元素必须不可变**，Python的基本类型如字符串、整数、浮点数都是不可变的，都可以作为 key。但是list是可变的，就不能作为 key。

  ```python
  d = {
      'Adam': 95,
      'Lisa': 85,
      'Bart': 59
  }

  # 计算集合长度
  print len(d)

  # None
  print d.get('Paul')

  # KeyError: 'Paul'
  if 'Paul' in d:
      print d['Paul']
  ```

  1. 访问dict:

     一是使用 d[key] 的形式来查找对应的 value，这和 list 很像，不同之处是，**list 必须使用索引返回对应的元素，而dict使用key**。如果key不存在，会直接报错：KeyError。

     二是使用dict本身提供的一个 get 方法，在Key不存在的时候，返回None。

  2. 更新dict：`d['Paul'] = 72`如果 key 已经存在，则赋值会用新的 value 替换掉原来的 value

  3. 遍历dict：通过`for key in d:`

  4. 删除元素：可以选择两种方式, `dict.pop(key)`和`del dict[key]`.

  **dict对象**本身就是可**迭代对象**，用 for 循环直接迭代 dict，可以每次拿到dict的一个key。

  如果希望迭代 dict 对象的value，应该怎么做？

  dict 对象有一个 **values() 方法**，这个方法把dict转换成一个包含所有value的list，这样，我们迭代的就是 dict的每一个 value：

  ```python
  d = { 'Adam': 95, 'Lisa': 85, 'Bart': 59 }
  print d.values()
  # [85, 95, 59]
  for v in d.values():
      print v
  # 85
  # 95
  # 59
  ```

  如果仔细阅读Python的文档，还可以发现，dict除了**values()**方法外，还有一个 `itervalues()` 方法，用 `itervalues()`方法替代 **values()** 方法，迭代效果完全一样：

  ```python
  d = { 'Adam': 95, 'Lisa': 85, 'Bart': 59 }
  print d.itervalues()
  # <dictionary-valueiterator object at 0x106adbb50>
  for v in d.itervalues():
      print v
  # 85
  # 95
  # 59
  ```

  **那这两个方法有何不同之处呢？**

  1.  **values()** 方法实际上把一个 dict 转换成了包含 value 的list。
  2. 但是 **itervalues()** 方法不会转换，它会在迭代过程中依次从 dict 中取出 value，所以 itervalues() 方法比 values() 方法节省了生成 list 所需的内存。
  3. 打印 itervalues() 发现它返回一个 `<dictionary-valueiterator>` 对象，这说明在Python中，**for 循环可作用的迭代对象远不止 list，tuple，str，unicode，dict等**，任何可迭代对象都可以作用于for循环，而内部如何迭代我们通常并不用关心。

  **如果一个对象说自己可迭代，那我们就直接用 for 循环去迭代它，可见，迭代是一种抽象的数据操作，它不对迭代对象内部的数据有任何要求。**

  在一个 for 循环中，能否同时迭代 key和value？答案是肯定的。

  首先，看看 dict 对象的 **items()** 方法返回的值：

  ```python
  >>> d = { 'Adam': 95, 'Lisa': 85, 'Bart': 59 }
  >>> print d.items()
  [('Lisa', 85), ('Adam', 95), ('Bart', 59)]
  ```

  可以看到，items() 方法把dict对象转换成了包含tuple的list，对这个list进行迭代，可以同时获得key和value：

  ```python
  >>> for key, value in d.items():
  ...     print key, ':', value
  ... 
  Lisa : 85
  Adam : 95
  Bart : 59
  ```

  和 values() 有一个 itervalues() 类似， **items() **也有一个对应的 **iteritems()**，iteritems() 不把dict转换成list，而是在迭代过程中不断给出 tuple，所以， iteritems() 不占用额外的内存。

- set **持有一系列元素，这一点和 list 很像，但是set的元素没有重复，而且是无序的，这点和 dict 的 key很像**。

  1. 创建：调用 set() 并传入一个 list，list的元素将作为set的元素

     ```python
     s = set(['A', 'B', 'C'])

     # 添加元素
     s.add(3)
     ```

  2. 更新：添加元素时，用set的add()方法，如果添加的元素已经存在于set中，add()不会报错

  3. 删除：删除set中的元素时，用set的remove()方法，如果删除的元素不存在set中，remove()会报错

     用add()可以直接添加，而remove()前需要判断。


##七、列表生成式

要生成list [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]，可以用`range(1, 11)`：

```python
>>> range(1, 11)
[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

但如果要生成[1x1, 2x2, 3x3, ..., 10x10]怎么做？列表生成式则可以用一行语句代替循环生成上面的list：

```python
>>> [x * x for x in range(1, 11)]
[1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
```

这种写法就是Python特有的列表生成式。利用列表生成式，可以以非常简洁的代码生成 list。

写列表生成式时，把要生成的元素 x * x 放到前面，后面跟 for 循环，就可以把list创建出来，十分有用，多写几次，很快就可以熟悉这种语法。

使用**for循环**的迭代不仅可以迭代普通的list，还可以迭代dict。

假设有如下的dict：

```python
d = { 'Adam': 95, 'Lisa': 85, 'Bart': 59 }
```

完全可以通过一个复杂的列表生成式把它变成一个 HTML 表格：

```python
tds = ['<tr><td>%s</td><td>%s</td></tr>' % (name, score) for name, score in d.iteritems()]
print '<table>'
print '<tr><th>Name</th><th>Score</th><tr>'
print '\n'.join(tds)
print '</table>'

# 输出内容
<table border="1">
<tr><th>Name</th><th>Score</th><tr>
<tr><td>Lisa</td><td>85</td></tr>
<tr><td>Adam</td><td>95</td></tr>
<tr><td>Bart</td><td>59</td></tr>
</table>
```

**注：**字符串可以通过 % 进行格式化，用指定的参数替代** **%s。字符串的join()方法可以把一个 list 拼接成一个字符串。

### 条件过滤

列表生成式的 **for 循环后面还可以加上 if 判断**。例如：

```python
>>> [x * x for x in range(1, 11)]
[1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
```

如果我们只想要偶数的平方，不改动 range()的情况下，可以加上 if 来筛选：

```python
>>> [x * x for x in range(1, 11) if x % 2 == 0]
[4, 16, 36, 64, 100]
```

有了 if 条件，只有 if 判断为 True 的时候，才把循环的当前元素添加到列表中。

### 多层表达式

for循环可以嵌套，因此，在列表生成式中，也可以用多层 for 循环来生成列表。

对于字符串 'ABC' 和 '123'，可以使用两层循环，生成全排列：

```python
>>> [m + n for m in 'ABC' for n in '123']
['A1', 'A2', 'A3', 'B1', 'B2', 'B3', 'C1', 'C2', 'C3']
```

翻译成循环代码就像下面这样：

```python
L = []
for m in 'ABC':
    for n in '123':
        L.append(m + n)
```

## 八、切片

对集合取指定索引范围的操作，用循环十分繁琐，因此，Python提供了切片（Slice）操作符，能大大简化这种操作。如取集合前3个元素，用一行代码就可以完成切片：

```python
>>> L[0:3]
['Adam', 'Lisa', 'Bart']
```

`L[0:3]`表示，从索引0开始取，直到索引3为止，==但不包括索引3==。即索引0，1，2，正好是3个元素。

如果第一个索引是0，还可以省略：

```python
>>> L[:3]
['Adam', 'Lisa', 'Bart']
```

也可以从索引1开始，取出2个元素出来：

```python
>>> L[1:3]
['Lisa', 'Bart']
```

只用一个` : `，表示从头到尾，因此，`L[:]`实际上复制出了一个新list：

```python
>>> L[:]
['Adam', 'Lisa', 'Bart', 'Paul']
```

切片操作还可以指定第三个参数，第三个参数表示每N个取一个，上面的` L[::2] `会每两个元素取出一个来，也就是隔一个取一个：

```python
>>> L[::2]
['Adam', 'Bart']
```

把list换成tuple，切片操作完全相同，只是切片的结果也变成了tuple。

###倒序切片

对于list，既然Python支持`L[-1]`取倒数第一个元素，那么它同样支持倒数切片，试试：

```python
>>> L = ['Adam', 'Lisa', 'Bart', 'Paul']

>>> L[-2:]
['Bart', 'Paul']

>>> L[:-2]
['Adam', 'Lisa']

>>> L[-3:-1]
['Lisa', 'Bart']

>>> L[-4:-1:2]
['Adam', 'Bart']
```

记住倒数第一个元素的索引是-1。倒序切片包含起始索引，不包含结束索引。

### 字符串切片

字符串 'xxx'和 Unicode字符串 u'xxx'也可以看成是一种list，每个元素就是一个字符。因此，字符串也可以用切片操作，只是操作结果仍是字符串：

```python
>>> 'ABCDEFG'[:3]
'ABC'
>>> 'ABCDEFG'[-3:]
'EFG'
>>> 'ABCDEFG'[::2]
'ACEG'
```

在很多编程语言中，针对字符串提供了很多各种截取函数，其实目的就是对字符串切片。Python没有针对字符串的截取函数，只需要切片一个操作就可以完成，非常简单。

