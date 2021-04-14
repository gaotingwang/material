[TOC]

## 函数式编程

![函数式编程概念](https://gtw.oss-cn-shanghai.aliyuncs.com/python/%E5%87%BD%E6%95%B0%E5%BC%8F%E7%BC%96%E7%A8%8B%E6%A6%82%E5%BF%B5.png)

函数式编程的特点:

- 把计算视为函数而非指令(这样不贴近计算机而是贴进计算)
- 纯函数式编程: 不需要变量，没有副作用，测试简单。(函数任意执行多少次结果确定)
- 支持高阶函数，代码简洁

python支持的函数式编程

- 不是纯函数式编程：允许有变量
- 支持高阶函数：函数也可以作为变量传入
- 支持闭包：有了闭包就能返回函数
- 有限度的支持匿名函数。

**高阶函数高在自己能接受函数作为参数**

### python中的高阶函数

#### map()函数

`map()`是 Python 内置的高阶函数，它接收一个**函数 f** 和一个 **list**，并通过把函数 f 依次作用在 list 的每个元素上，得到一个新的 list 并返回。

例如，对于list [1, 2, 3, 4, 5, 6, 7, 8, 9]

如果希望把list的每个元素都作平方，就可以用map()函数：

![map()函数](http://img.mukewang.com/54c8a7e40001327303410245.png)

因此，我们只需要传入函数f(x)=x*x，就可以利用map()函数完成这个计算：

```python
def f(x):
    return x*x
print map(f, [1, 2, 3, 4, 5, 6, 7, 8, 9])
```

**输出结果：**

```
[1, 4, 9, 10, 25, 36, 49, 64, 81]
```

**注意：**map()函数不改变原有的 list，而是返回一个新的 list。

利用map()函数，可以把一个 list 转换为另一个 list，只需要传入转换函数。

由于list包含的元素可以是任何类型，因此，map() 不仅仅可以处理只包含数值的 list，事实上它可以处理包含任意类型的 list，只要传入的函数f可以处理这种数据类型。

#### reduce()函数

`reduce()`函数也是Python内置的一个高阶函数。reduce()函数接收的参数和 map()类似，**一个函数 f，一个list**，但行为和 map()不同，==reduce()传入的函数 f 必须接收两个参数==，reduce()对list的每个元素反复调用函数f，并返回最终结果值。

例如，编写一个f函数，接收x和y，返回x和y的和：

```python
def f(x, y):
    return x + y
```

调用 **reduce(f, [1, 3, 5, 7, 9])**时，reduce函数将做如下计算：

```
先计算头两个元素：f(1, 3)，结果为4；
再把结果和第3个元素计算：f(4, 5)，结果为9；
再把结果和第4个元素计算：f(9, 7)，结果为16；
再把结果和第5个元素计算：f(16, 9)，结果为25；
由于没有更多的元素了，计算结束，返回结果25。
```

上述计算实际上是对 list 的所有元素求和。虽然Python内置了求和函数sum()，但是，利用reduce()求和也很简单。

**reduce()还可以接收第3个可选参数，作为计算的初始值。**如果把初始值设为100，计算：

```python
reduce(f, [1, 3, 5, 7, 9], 100)
```

结果将变为125，因为第一轮计算是：

计算初始值和第一个元素：**f(100, 1)**，结果为**101**。

#### filter()函数

`filter()`函数是 Python 内置的另一个有用的高阶函数，filter()函数接收一个**函数 f **和一个**list**，这个函数 f 的作用是对每个元素进行判断，返回 True或 False，**filter()根据判断结果自动过滤掉不符合条件的元素，返回由符合条件元素组成的新list。**

例如，要从一个list [1, 4, 6, 7, 9, 12, 17]中删除偶数，保留奇数，首先，要编写一个判断奇数的函数：

```python
def is_odd(x):
    return x % 2 == 1
```

然后，利用filter()过滤掉偶数：

```python
filter(is_odd, [1, 4, 6, 7, 9, 12, 17])
```

**结果：**[1, 7, 9, 17]

利用filter()，可以完成很多有用的功能，例如，删除 None 或者空字符串：

```python
def is_not_empty(s):
    return s and len(s.strip()) > 0
filter(is_not_empty, ['test', None, '', 'str', '  ', 'END'])
```

**结果：**['test', 'str', 'END']

####sorted()函数

Python内置的 **sorted()**函数可对list进行排序：

```python
>>>sorted([36, 5, 12, 9, 21])
[5, 9, 12, 21, 36]
```

但 **sorted()**也是一个高阶函数，它**可以接收一个比较函数来实现自定义排序**，比较函数的定义是，传入两个待比较的元素 x, y，==如果 x 应该排在 y 的前面，返回 -1，如果 x 应该排在 y 的后面，返回 1。如果 x 和 y 相等，返回 0。==

因此，如果要实现倒序排序，只需要编写一个reversed_cmp函数：

```python
def reversed_cmp(x, y):
    if x > y:
        return -1
    if x < y:
        return 1
    return 0
```

这样，调用 sorted() 并传入 reversed_cmp 就可以实现倒序排序：

```python
>>> sorted([36, 5, 12, 9, 21], reversed_cmp)
[36, 21, 12, 9, 5]
```

sorted()也可以对字符串进行排序，字符串默认按照ASCII大小来比较：

```python
>>> sorted(['bob', 'about', 'Zoo', 'Credit'])
['Credit', 'Zoo', 'about', 'bob']
```

### 返回函数

Python的函数不但可以返回int、str、list、dict等数据类型，还可以返回函数！

例如，定义一个函数 f()，我们让它返回一个函数 g，可以这样写：

```python
def f():
    print 'call f()...'
    # 定义函数g:
    def g():
        print 'call g()...'
    # 返回函数g:
    return g
```

仔细观察上面的函数定义，我们在函数 f 内部又定义了一个函数 g。由于函数 g 也是一个对象，函数名 g 就是指向函数 g 的变量，所以，最外层函数 f 可以返回变量 g，也就是函数 g 本身。

调用函数 f，我们会得到 f 返回的一个函数：

```python
>>> x = f()   # 调用f()
call f()...
>>> x   # 变量x是f()返回的函数：
<function g at 0x1037bf320>
>>> x()   # x指向函数，因此可以调用
call g()...   # 调用x()就是执行g()函数定义的代码
```

请注意区分返回函数和返回值：

```python
def myabs():
    return abs   # 返回函数
def myabs2(x):
    return abs(x)   # 返回函数调用的结果，返回值是一个数值
```

返回函数可以把一些计算延迟执行。例如，如果定义一个普通的求和函数：

```python
def calc_sum(lst):
    return sum(lst)
```

调用calc_sum()函数时，将立刻计算并得到结果：

```python
>>> calc_sum([1, 2, 3, 4])
10
```

但是，如果返回一个函数，就可以“延迟计算”：

```python
def calc_sum(lst):
    def lazy_sum():
        return sum(lst)
    return lazy_sum
```

\# 调用calc_sum()并没有计算出结果，而是返回函数:

```python
>>> f = calc_sum([1, 2, 3, 4])
>>> f
<function lazy_sum at 0x1037bfaa0>
```

\# 对返回的函数进行调用时，才计算出结果:

```python
>>> f()
10
```

由于可以返回函数，我们在后续代码里就可以决定到底要不要调用该函数。

###python中闭包

在函数内部定义的函数和外部定义的函数是一样的，只是他们无法被外部访问：

```python
def g():
    print 'g()...'

def f():
    print 'f()...'
    return g
```

将**g**的定义移入函数 **f** 内部，防止其他代码调用 **g**：

```python
def f():
    print 'f()...'
    def g():
        print 'g()...'
    return g
```

但是，考察上一小节定义的 **calc_sum **函数：

```python
def calc_sum(lst):
    def lazy_sum():
        return sum(lst)
    return lazy_sum
```

**注意: **发现没法把 **lazy_sum** 移到 **calc_sum** 的外部，因为它引用了 **calc_sum** 的参数 **lst**。

==像这种内层函数引用了外层函数的变量（参数也算变量），然后返回内层函数的情况，称为**闭包（Closure）**。==

==**闭包的特点**是返回的函数还引用了外层函数的局部变量，所以，要正确使用闭包，就要确保引用的局部变量在函数返回后不能变。==举例如下：

```python
# 希望一次返回3个函数，分别计算1x1,2x2,3x3:
def count():
    fs = []
    for i in range(1, 4):
        def f():
             return i*i
        fs.append(f)
    return fs

f1, f2, f3 = count()
```

你可能认为调用f1()，f2()和f3()结果应该是1，4，9，但实际结果全部都是 9。

原因就是当count()函数返回了3个函数时，这3个函数所引用的变量 i 的值已经变成了3。由于f1、f2、f3并没有被调用，所以，此时他们并未计算 i*i，当 f1 被调用时：

```python
>>> f1()
9     # 因为f1现在才计算i*i，但现在i的值已经变为3
```

因此，返回函数不要引用任何循环变量，或者后续会发生变化的变量。

解决方案：

```python
def count():
    fs = []
    for i in range(1, 4):
        def f(j):
            def g():
                return j*j
            return g
        r = f(i) # 用变量指向当前函数，然后放入数组中
        fs.append(r)
    return fs
f1, f2, f3 = count()
print f1(), f2(), f3()
```

### 匿名函数

高阶函数可以接收函数做参数，有些时候，我们不需要显式地定义函数，直接传入匿名函数更方便。

在Python中，对匿名函数提供了有限支持。还是以map()函数为例，计算 f(x)=x^2^ 时，除了定义一个f(x)的函数外，还可以直接传入匿名函数：

```python
>>> map(lambda x: x * x, [1, 2, 3, 4, 5, 6, 7, 8, 9])
[1, 4, 9, 16, 25, 36, 49, 64, 81]
```

通过对比可以看出，匿名函数 lambda x: x * x 实际上就是：

```python
def f(x):
    return x * x
```

==关键字lambda 表示匿名函数，冒号前面的 x 表示函数参数。==

==匿名函数有个限制，就是**只能有一个表达式**，**不写return**，返回值就是该表达式的结果。==

使用匿名函数，可以不必定义函数名，直接创建一个函数对象，很多时候可以简化代码：

```python
>>> sorted([1, 3, 9, 5, 0], lambda x,y: -cmp(x,y))
[9, 5, 3, 1, 0]
```

返回函数的时候，也可以返回匿名函数：

```python
>>> myabs = lambda x: -x if x < 0 else x # 类似于三元表达式 myabs = x < 0 ? -x : x
>>> myabs(-1)
1
>>> myabs(1)
1
```

## 装饰器Decorator

在不改变原函数的情况下，想在运行时动态的增加新功能。可以极大地简化代码，避免每个函数编写重复性代码。

通过高阶函数返回新函数：

```python
def f1(x):
    return x*2
  
def new_fn(f):
    def fn(x):
        print 'call' + f.__name__ + '()'
        return f(x)
    return fn
```

Python内置的`@`语法就是为了简化装饰器的调用。

```python
@new_fn
def f1(x):
    return x*2
```

Python的 decorator 本质上就是一个高阶函数，它接收一个函数作为参数，然后，返回一个新函数。使用 decorator 用Python提供的 @ 语法，这样可以避免手动编写 f = decorate(f) 这样的代码。

**考察一个@log的定义：**

```python
def log(f):
    def fn(x):
        print 'call ' + f.__name__ + '()...'
        return f(x)
    return fn
```

对于阶乘函数，@log工作得很好：

```python
@log
def factorial(n):
    return reduce(lambda x,y: x*y, range(1, n+1))
print factorial(10)
```

**结果：**

```
call factorial()...
3628800
```

但是，对于参数不是一个的函数，调用将报错：

```python
@log
def add(x, y):
    return x + y
print add(1, 2)
```

**结果：**

```python
Traceback (most recent call last):
  File "test.py", line 15, in <module>
    print add(1,2)
TypeError: fn() takes exactly 1 argument (2 given)
```

因为 add() 函数需要传入两个参数，但是 @log 写死了只含一个参数的返回函数。

要让 @log 自适应任何参数定义的函数，可以利用Python的 *args 和 **kw，保证任意个数的参数总是能正常调用：

```python
def log(f):
    def fn(*args, **kw):
        print 'call ' + f.__name__ + '()...'
        return f(*args, **kw)
    return fn
```

现在，对于任意函数，@log 都能正常工作。

###带参数的Decorator

log函数本身就需要传入'INFO'或'DEBUG'这样的参数，类似这样：

```python
@log('DEBUG')
def my_func():
    pass
```

带参数的log函数首先返回一个decorator函数，再让这个decorator函数接收my_func并返回新函数：

```python
def log(prefix):
    def log_decorator(f):
        def wrapper(*args, **kw):
            print '[%s] %s()...' % (prefix, f.__name__)
            return f(*args, **kw)
        return wrapper
    return log_decorator

@log('DEBUG')
def test():
    pass
print test()
```

**执行结果：**

```
[DEBUG] test()...
None
```

对于这种3层嵌套的decorator定义，你可以先把它拆开：

```python
# 标准decorator:
def log_decorator(f):
    def wrapper(*args, **kw):
        print '[%s] %s()...' % (prefix, f.__name__)
        return f(*args, **kw)
    return wrapper
return log_decorator

# 返回decorator:
def log(prefix):
    return log_decorator(f)
```

拆开以后会发现，调用会失败，因为在3层嵌套的decorator定义中，最内层的wrapper引用了最外层的参数prefix，所以，把一个闭包拆成普通的函数调用会比较困难。不支持闭包的编程语言要实现同样的功能就需要更多的代码。

### 完善Decorator

如果要让调用者看不出一个函数经过了@decorator的“改造”，就需要把原函数的一些属性复制到新函数中：

```python
def log(f):
    def wrapper(*args, **kw):
        print 'call...'
        return f(*args, **kw)
    wrapper.__name__ = f.__name__
    wrapper.__doc__ = f.__doc__
    return wrapper
```

这样写decorator很不方便，因为我们也很难把原函数的所有必要属性都一个一个复制到新函数上，所以==Python内置的functools可以用来自动化完成这个“复制”的任务==：

```python
import functools
def log(f):
    @functools.wraps(f)
    def wrapper(*args, **kw):
        print 'call...'
        return f(*args, **kw)
    return wrapper
```

最后需要指出，由于我们把原函数签名改成了(*args, **kw)，因此，无法获得原函数的原始参数信息。即便我们采用固定参数来装饰只有一个参数的函数：

```python
def log(f):
    @functools.wraps(f)
    def wrapper(x):
        print 'call...'
        return f(x)
    return wrapper
```

也可能改变原函数的参数名，因为新函数的参数名始终是 'x'，原函数定义的参数名不一定叫 'x'。

### 偏函数

int()函数可以把字符串转换为整数，当仅传入字符串时，int()函数默认按十进制转换：

```python
>>> int('12345')
12345
```

但int()函数还提供额外的base参数，默认值为10。如果传入base参数，就可以做 N 进制的转换：

```python
>>> int('12345', base=8)
5349
>>> int('12345', 16)
74565
```

假设要转换大量的二进制字符串，每次都传入int(x, base=2)非常麻烦，于是，我们想到，可以定义一个int2()的函数，默认把base=2传进去：

```python
def int2(x, base=2):
    return int(x, base)
```

这样，我们转换二进制就非常方便了：

```python
>>> int2('1000000')
64
>>> int2('1010101')
85
```

functools.partial就是帮助我们创建一个偏函数的，不需要我们自己定义int2()，可以直接使用下面的代码创建一个新的函数int2：

```python
>>> import functools
>>> int2 = functools.partial(int, base=2)
>>> int2('1000000')
64
>>> int2('1010101')
85
```

所以，functools.partial可以把一个参数多的函数变成一个参数少的新函数，少的参数需要在创建时指定默认值，这样，新函数调用的难度就降低了。

## 模块使用

- 包就是文件夹
- 模块就是`xxx.py`文件
- 包可以有多级

区分包和普通目录:==包下面必须有一个`__init__.py`这样一个特殊的文件。每一个包的**每一层目录**都要有这个文件。即使这个文件是空文件，也必须让这个文件存在。这样Python才能识别这是一个包。==

### 导入模块

要使用一个模块，我们必须首先导入该模块。Python使用import语句导入一个模块。例如，导入系统自带的模块 math：

```python
import math
```

你可以认为math就是一个指向已导入模块的变量，通过该变量，我们可以访问math模块中所定义的所有公开的函数、变量和类：

```python
>>> math.pow(2, 0.5) # pow是函数
1.4142135623730951

>>> math.pi # pi是变量
3.141592653589793
```

如果只希望导入用到的math模块的某几个函数，而不是所有函数，可以用下面的语句：

```python
from math import pow, sin, log
```

这样，可以直接引用 pow, sin, log 这3个函数，但math的其他函数没有导入进来：

```python
>>> pow(2, 10)
1024.0
>>> sin(3.14)
0.0015926529164868282
```

如果遇到名字冲突怎么办？比如math模块有一个log函数，logging模块也有一个log函数，如果同时使用，如何解决名字冲突？

如果使用import导入模块名，由于必须通过模块名引用函数名，因此不存在冲突：

```python
import math, logging
print math.log(10)   # 调用的是math的log函数
logging.log(10, 'something')   # 调用的是logging的log函数
```

如果使用 from...import 导入 log 函数，势必引起冲突。这时，可以给函数起个“别名”来避免冲突：

```python
from math import log
from logging import log as logger   # logging的log现在变成了logger
print log(10)   # 调用的是math的log
logger(10, 'import from logging')   # 调用的是logging的log
```

利用ImportError错误，经常在Python中动态导入模块：

```python
try:
    from cStringIO import StringIO
except ImportError:
    from StringIO import StringIO
```

上述代码先尝试从cStringIO导入，如果失败了（比如cStringIO没有被安装），再尝试从StringIO导入。这样，如果cStringIO模块存在，则我们将获得更快的运行速度，如果cStringIO不存在，则顶多代码运行速度会变慢，但不会影响代码的正常执行。

**try **的作用是捕获错误，并在捕获到指定错误时执行 **except **语句。

### 使用`__future__`

Python的新版本会引入新的功能，但是，实际上这些功能在上一个老版本中就已经存在了。要“试用”某一新的特性，就可以通过导入__future__模块的某些功能来实现。

例如，Python 2.7的整数除法运算结果仍是整数，Python 3.x已经改进了整数的除法运算，“**/**”除将得到浮点数，“**//**”除才仍是整数，要在Python 2.7中引入3.x的除法规则，导入**__future__**的**division**：

```python
>>> from __future__ import division
>>> print 10 / 3
3.3333333333333335
```

### 第三方模块安装

一般会使用pip3,这是一个安装模块非常有利的工具，例如进行numpy模块安装：

```shell
sudo pip3 install numpy
sudo pip3 install -U numpy 升级模块
```

## 面向对象编程

在Python中，类通过 **class **关键字定义。以 **Person** 为例，定义一个**Person类**如下：

```python
class Person(object):
    pass
```

按照 **Python** 的编程习惯，类名以大写字母开头，紧接着是(object)，表示该类是从哪个类继承下来的。类的继承将在后面的章节讲解，现在我们只需要简单地从**object**类继承。

有了Person类的定义，就可以创建出具体的**xiaoming、xiaohong**等实例。创建实例使用 **类名+()**，类似函数调用的形式创建：

```
xiaoming = Person()
xiaohong = Person()
```

如何让每个实例拥有各自不同的属性？==由于Python是动态语言，对每一个实例，都可以直接给他们的属性赋值==，例如，给**xiaoming**这个实例加上**name、gender**和**birth**属性：

```python
xiaoming = Person()
xiaoming.name = 'Xiao Ming'
xiaoming.gender = 'Male'
xiaoming.birth = '1990-1-1'
```

给**xiaohong**加上的属性不一定要和**xiaoming**相同：

```python
xiaohong = Person()
xiaohong.name = 'Xiao Hong'
xiaohong.school = 'No. 1 High School'
xiaohong.grade = 2
```

实例的属性可以像普通变量一样进行操作：

```
xiaohong.grade = xiaohong.grade + 1
```

### 初始化实例属性

虽然可以自由地给一个实例绑定各种属性，但是，现实世界中，一种类型的实例应该拥有相同名字的属性。例如，**Person类**应该在创建的时候就拥有 **name、gender **和 **birth **属性，怎么办？

在定义 Person 类时，可以为Person类添加一个特殊的`__init__()`方法，当创建实例时，**`__init__()`**方法被自动调用，就能在此为每个实例都统一加上以下属性：

```python
class Person(object):
    def __init__(self, name, gender, birth):
        self.name = name
        self.gender = gender
        self.birth = birth
```

==**`__init__()` **方法的第一个参数必须是 **self**==（也可以用别的名字，但建议使用习惯用法），后续参数则可以自由指定，和定义函数没有任何区别。

相应地，创建实例时，就必须要提供除 **self **以外的参数：

```
xiaoming = Person('Xiao Ming', 'Male', '1991-1-1')
xiaohong = Person('Xiao Hong', 'Female', '1992-2-2')

```

有了**__init__()**方法，每个Person实例在创建时，都会有 **name、gender **和 **birth **这3个属性，并且，被赋予不同的属性值，访问属性使用.操作符：

```
print xiaoming.name
# 输出 'Xiao Ming'
print xiaohong.birth
# 输出 '1992-2-2'

```

要特别注意的是，初学者定义**`__init__()`**方法常常忘记了 self 参数，第一个参数会被Python解释器传入了实例的引用。

### 类属性

定义类属性可以直接在 **class **中定义：

```python
class Person(object):
    address = 'Earth'
    def __init__(self, name):
        self.name = name
```

==因为类属性是直接绑定在类上的，所以，访问类属性不需要创建实例，就可以直接访问==：

```python
print Person.address
# => Earth
```

==对一个实例调用类的属性也是可以访问的，所有实例都可以访问到它所属的类的属性==：

```python
p1 = Person('Bob')
p2 = Person('Alice')
print p1.address
# => Earth
print p2.address
# => Earth
```

由于Python是动态语言，类属性也是可以动态添加和修改的：

```python
Person.address = 'China'
print p1.address
# => 'China'
print p2.address
# => 'China'
```

==因为类属性只有一份，所以，当**Person**类的**address**改变时，所有实例访问到的类属性都改变了==。

如果在实例变量上修改类属性会发生什么问题呢？

```python
p1.address = 'China'
print 'p1.address = ' + p1.address

print 'Person.address = ' + Person.address
print 'p2.address = ' + p2.address
```

结果如下：

```python
Person.address = Earth
p1.address = China
Person.address = Earth
p2.address = Earth
```

原因是 p1.address = 'China'并没有改变 Person 的 address，而是给 p1这个实例绑定了实例属性address，对p1来说，它有一个实例属性address（值是'China'），而它所属的类Person也有一个类属性address，所以:

访问 p1.address 时，优先查找实例属性，返回'China'。

访问 p2.address 时，p2没有实例属性address，但是有类属性address，因此返回'Earth'。

可见，==当实例属性和类属性重名时，实例属性优先级高，它将屏蔽掉对类属性的访问==。

当我们把 p1 的 address 实例属性删除后，访问 p1.address 就又返回类属性的值 'Earth'了：

```python
del p1.address
print p1.address
# => Earth
```

可见，千万不要在实例上修改类属性，它实际上并没有修改类属性，而是给实例绑定了一个实例属性。

### 访问限制

我们可以给一个实例绑定很多属性，如果有些属性不希望被外部访问到怎么办？

Python对属性权限的控制是通过属性名来实现的，==如果一个属性由双下划线开头(__)，该属性就无法被外部访问==。看例子：

```python
class Person(object):
    def __init__(self, name):
        self.name = name
        self._title = 'Mr'
        self.__job = 'Student'
p = Person('Bob')
print p.name
# => Bob
print p._title
# => Mr
print p.__job
# => Error
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'Person' object has no attribute '__job'
```

可见，只有以双下划线开头的**`__job`**不能直接被外部访问。

但是，如果一个属性以`__xxx__`的形式定义，那它又可以被外部访问了，以`__xxx__`定义的属性在Python的类中被称为特殊属性，有很多预定义的特殊属性可以使用，通常不要把普通属性用`__xxx__`定义。

以单下划线开头的属性"_xxx"虽然也可以被外部访问，但是，按照习惯，他们不应该被外部访问。

### 定义实例方法

**实例的方法**就是在类中定义的函数，它的==第一个参数永远是 self==，指向调用该方法的实例本身，其他参数和一个普通函数是完全一样的：

```python
class Person(object):

    def __init__(self, name):
        self.__name = name

    def get_name(self):
        return self.__name
```

**get_name(self) **就是一个实例方法，它的第一个参数是self。`__init__(self, name)`其实也可看做是一个特殊的实例方法。

调用实例方法必须在实例上调用：

```python
p1 = Person('Bob')
print p1.get_name()  # self不需要显式传入
# => Bob
```

在实例方法内部，可以访问所有实例属性，这样，如果外部需要访问私有属性，可以通过方法调用获得，这种数据封装的形式除了能保护内部数据一致性外，还可以简化外部调用的难度。

==在 **class** 中定义的实例方法其实也是属性，它实际上是一个函数对象==

因为方法也是一个属性，所以，它也可以动态地添加到实例上，只是需要用 types.MethodType() 把一个函数变为一个方法：

```python
import types
def fn_get_grade(self):
    if self.score >= 80:
        return 'A'
    if self.score >= 60:
        return 'B'
    return 'C'

class Person(object):
    def __init__(self, name, score):
        self.name = name
        self.score = score

p1 = Person('Bob', 90)
p1.get_grade = types.MethodType(fn_get_grade, p1, Person)
print p1.get_grade()
# => A
p2 = Person('Alice', 65)
print p2.get_grade()
# ERROR: AttributeError: 'Person' object has no attribute 'get_grade'
# 因为p2实例并没有绑定get_grade
```

给一个实例动态添加方法并不常见，直接在class中定义要更直观。

### 定义类方法

要在class中定义类方法，需要这么写：

```python
class Person(object):
    count = 0
    @classmethod
    def how_many(cls):
        return cls.count
    def __init__(self, name):
        self.name = name
        Person.count = Person.count + 1

print Person.how_many()
p1 = Person('Bob')
print Person.how_many()
```

==通过标记一个 `@classmethod`，该方法将绑定到 Person 类上，而非类的实例==。==类方法的第一个参数将传入类本身，通常将参数名命名为**cls**==，上面的 **cls.count **实际上相当于 **Person.count**。

因为是在类上调用，而非实例上调用，因此类方法无法获得任何实例变量，只能获得类的引用。

### 继承类

```python
class Person(object):
    def __init__(self, name, gender):
        self.name = name
        self.gender = gender
```

定义**Student**类时，只需要把额外的属性加上，例如score：

```python
class Student(Person):
    def __init__(self, name, gender, score):
        super(Student, self).__init__(name, gender)
        self.score = score
```

一定要==用 `super(Student, self).__init__(name, gender)` 去初始化父类==，否则，继承自 **Person** 的 **Student** 将没有**name** 和 **gender**。

==函数**super(Student, self)**将返回当前类继承的父类==，即Person，然后调用`__init__()`方法，注意self参数已在super()中传入，在`__init__()`中将隐式传递，不需要写出（也不能写）。

==函数**isinstance()**可以判断一个变量的类型==，既可以用在Python内置的数据类型如**str、list、dict**，也可以用在我们自定义的类，它们本质上都是数据类型。

```python
>>> isinstance(p, Person)
True    # p是Person类型
>>> isinstance(p, Student)
False   # p不是Student类型
>>> isinstance(p, Teacher)
False   # p不是Teacher类型
```

Python允许从多个父类继承，称为多重继承，例如，**D **同时继承自 **B** 和 **C**:

```python
class D(B, C):
    def __init__(self, a):
        super(D, self).__init__(a)
        print 'init D...'
```

### 多态

类具有继承关系，并且子类类型可以向上转型看做父类类型，如果我们从 **Person **派生出 **Student**和**Teacher **，并都写了一个 **whoAmI() **方法：

```python
class Person(object):
    def __init__(self, name, gender):
        self.name = name
        self.gender = gender
    def whoAmI(self):
        return 'I am a Person, my name is %s' % self.name

class Student(Person):
    def __init__(self, name, gender, score):
        super(Student, self).__init__(name, gender)
        self.score = score
    def whoAmI(self):
        return 'I am a Student, my name is %s' % self.name

class Teacher(Person):
    def __init__(self, name, gender, course):
        super(Teacher, self).__init__(name, gender)
        self.course = course
    def whoAmI(self):
        return 'I am a Teacher, my name is %s' % self.name
```

在一个函数中，如果我们接收一个变量**x**，则无论该 **x **是**Person、Student**还是 **Teacher**，都可以正确打印出结果：

```python
def who_am_i(x):
    print x.whoAmI()

p = Person('Tim', 'Male')
s = Student('Bob', 'Male', 88)
t = Teacher('Alice', 'Female', 'English')

who_am_i(p)
who_am_i(s)
who_am_i(t)
```

运行结果：

```
I am a Person, my name is Tim
I am a Student, my name is Bob
I am a Teacher, my name is Alice
```

由于Python是动态语言，所以，传递给函数 **who_am_i(x)**的参数 **x** 不一定是 Person 或 Person 的子类型。任何数据类型的实例都可以，只要它**有一个whoAmI()**的方法即可：

```
class Book(object):
    def whoAmI(self):
        return 'I am a book'
```

这是动态语言和静态语言（例如Java）最大的差别之一。==动态语言调用实例方法，**不检查类型**，只要方法存在，参数正确，就可以调用==。

### 获取对象信息

除了用 **isinstance() **判断它是否是某种类型的实例外，还有没有别的方法获取到更多的信息呢？

例如，已有定义：

首先可以用 **type() **函数获取变量的类型，它返回一个 **Type **对象：

```python
>>> type(123)
<type 'int'>
>>> s = Student('Bob', 'Male', 88)
>>> type(s)
<class '__main__.Student'>
```

其次，可以用 **dir() **函数获取变量的所有属性：

```python
>>> dir(123)   # 整数也有很多属性...
['__abs__', '__add__', '__and__', '__class__', '__cmp__', ...]

>>> dir(s)
['__class__', '__delattr__', '__dict__', '__doc__', '__format__', '__getattribute__', '__hash__', '__init__', '__module__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', 'gender', 'name', 'score', 'whoAmI']
```

对于实例变量，**dir()**返回所有实例属性，包括`__class__`这类有特殊意义的属性。注意到方法`whoAmI`也是 **s **的一个属性。

如何去掉**`__xxx__`**这类的特殊属性，只保留我们自己定义的属性？回顾一下**filter()**函数的用法。

**dir()**返回的属性是字符串列表，如果已知一个属性名称，要获取或者设置对象的属性，就需要用 **getattr() **和**setattr( )**函数了：

```python
>>> getattr(s, 'name')  # 获取name属性
'Bob'

>>> setattr(s, 'name', 'Adam')  # 设置新的name属性

>>> s.name
'Adam'

>>> getattr(s, 'age')  # 获取age属性，但是属性不存在，报错：
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'Student' object has no attribute 'age'

>>> getattr(s, 'age', 20)  # 获取age属性，如果属性不存在，就返回默认值20：
20
```

## 定制类

python的特殊方法特点：

- ==特殊方法定义在`class`中==
- ==不需要直接调用==
- ==python的某些函数或操作符会自动调用对应的特殊方法==

Python定义的一部分特殊方法:

![mark](http://myphoto.mtianyan.cn/blog/180104/25k9LCamf4.png?imageslim)

正确实现特殊方法：

- 只需编写用到的特殊方法
- 有关联性的特殊方法都必须实现

### `__str__`和`__repr__`

如果要把一个类的实例变成**str**，就需要实现特殊方法`__str__()`：

```python
class Person(object):
    def __init__(self, name, gender):
        self.name = name
        self.gender = gender
    def __str__(self):
        return '(Person: %s, %s)' % (self.name, self.gender)
```

现在，在交互式命令行下用 **print **试试：

```python
>>> p = Person('Bob', 'male')
>>> print p
(Person: Bob, male)
```

但是，如果直接敲变量 **p**：

```python
>>> p
<main.Person object at 0x10c941890>
```

似乎`__str__()` 不会被调用。

因为 Python 定义了`__str__()`和`__repr__()`两种方法，`__str__()`用于显示给用户，而`__repr__()`用于显示给开发人员。

有一个偷懒的定义`__repr__`的方法：

```python
class Person(object):
    def __init__(self, name, gender):
        self.name = name
        self.gender = gender
    def __str__(self):
        return '(Person: %s, %s)' % (self.name, self.gender)
    __repr__ = __str__
```

### `__cmp__`

Python的 **sorted() **按照默认的比较函数 **cmp **排序，但是，如果对一组 **Student **类的实例排序时，就必须提供我们自己的特殊方法 `__cmp__()`：

```python
class Student(object):
    def __init__(self, name, score):
        self.name = name
        self.score = score
    def __str__(self):
        return '(%s: %s)' % (self.name, self.score)
    __repr__ = __str__

    def __cmp__(self, s):
        if self.name < s.name:
            return -1
        elif self.name > s.name:
            return 1
        else:
            return 0
```

上述 Student 类实现了`__cmp__()`方法，`__cmp__`用实例自身**self**和传入的实例 **s **进行比较，如果**self** 应该排在前面，就返回 -1，如果 **s**应该排在前面，就返回1，如果两者相当，返回 0。

### 类型转换

要让**int()**函数正常工作，只需要实现特殊方法__int__():

```python
class Rational(object):
    def __init__(self, p, q):
        self.p = p
        self.q = q
    def __int__(self):
        return self.p // self.q
```

**结果如下：**

```python
>>> print int(Rational(7, 2))
3
>>> print int(Rational(1, 3))
0
```

同理，要让**float()**函数正常工作，只需要实现特殊方法**__float__()**。

### 数学运算

加法运算：`__add__`
减法运算：`__sub__`
乘法运算：`__mul__`
除法运算：`__div__`

### `@property`

考察 **Student **类：

```python
class Student(object):
    def __init__(self, name, score):
        self.name = name
        self.score = score
```

当我们想要修改一个**Student** 的 **scroe** 属性时，可以这么写：

```
s = Student('Bob', 59)
s.score = 60
```

但是也可以这么写：

```
s.score = 1000
```

显然，直接给属性赋值无法检查分数的有效性。

如果利用两个方法：

```python
class Student(object):
    def __init__(self, name, score):
        self.name = name
        self.__score = score
    def get_score(self):
        return self.__score
    def set_score(self, score):
        if score < 0 or score > 100:
            raise ValueError('invalid score')
        self.__score = score
```

这样一来，**s.set_score(1000)** 就会报错。

这种使用 **get/set **方法来封装对一个属性的访问在许多面向对象编程的语言中都很常见。

但是写**s.get_score()** 和 **s.set_score() **没有直接写 **s.score** 来得直接。

有没有两全其美的方法？----有。

因为Python支持高阶函数，在函数式编程中我们介绍了装饰器函数，可以用装饰器函数把 **get/set **方法“装饰”成属性调用：

```python
class Student(object):
    def __init__(self, name, score):
        self.name = name
        self.__score = score
    @property
    def score(self):
        return self.__score
    @score.setter
    def score(self, score):
        if score < 0 or score > 100:
            raise ValueError('invalid score')
        self.__score = score
```

**注意:** 第一个score(self)是get方法，用@property装饰，第二个score(self, score)是set方法，用@score.setter装饰，@score.setter是前一个@property装饰后的副产品。

现在，就可以像使用属性一样设置score了：

```python
>>> s = Student('Bob', 59)
>>> s.score = 60
>>> print s.score
60
>>> s.score = 1000
Traceback (most recent call last):
  ...
ValueError: invalid score
```

说明对**score **赋值实际调用的是 **set方法**。

### `__slots__`

由于Python是动态语言，任何实例在运行期都可以动态地添加属性。

如果要限制添加的属性，例如，**Student**类只允许添加 **name、gender**和**score **这3个属性，就可以利用Python的一个特殊的`__slots__`来实现。

顾名思义，`__slots__`是指一个类允许的属性列表：

```python
class Student(object):
    __slots__ = ('name', 'gender', 'score')
    def __init__(self, name, gender, score):
        self.name = name
        self.gender = gender
        self.score = score
```

现在，对实例进行操作：

```python
>>> s = Student('Bob', 'male', 59)
>>> s.name = 'Tim' # OK
>>> s.score = 99 # OK
>>> s.grade = 'A'
Traceback (most recent call last):
  ...
AttributeError: 'Student' object has no attribute 'grade'
```

`__slots__`的目的是限制当前类所能拥有的属性，如果不需要添加任意动态的属性，使用`__slots__`也能节省内存。

### `__call__`

在Python中，函数其实是一个对象：

```python
>>> f = abs
>>> f.__name__
'abs'
>>> f(-123)
123
```

由于 **f** 可以被调用，所以，**f** 被称为可调用对象。

所有的函数都是可调用对象。

一个类实例也可以变成一个可调用对象，只需要实现一个特殊方法`__call__()`。

我们把**Person **类变成一个可调用对象：

```python
class Person(object):
    def __init__(self, name, gender):
        self.name = name
        self.gender = gender

    def __call__(self, friend):
        print 'My name is %s...' % self.name
        print 'My friend is %s...' % friend
```

现在可以对**Person **实例直接调用：

```python
>>> p = Person('Bob', 'male')
>>> p('Tim')
My name is Bob...
My friend is Tim...
```

单看**p('Tim') **你无法确定 **p** 是一个函数还是一个类实例，所以，在Python中，函数也是对象，对象和函数的区别并不显著。

```python
class Fib(object):
    def __call__(self, num):
        a, b, L = 0, 1, []
        for n in range(num):
            L.append(a)
            a, b = b, a + b
        return L

f = Fib()
print f(10)

# 运行结果
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```



## 下一步学习

下一步进阶：

- IO：文件和Socket
- 多任务：进程和线程
- 数据库
- Web开发