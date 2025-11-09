---
title: "Python的模块和包"
date: 2025-09-19T14:52:00+08:00
summary: "为了不要把所有代码都堆在一个文件了，我们有了模块和包的概念"
---

## 目录

[TOC]

---

## 前言

模块和包的意义在于分解一个庞大而复杂的项目，构建起一个更加清晰，易于理解的结构。

将一长串代码分解成模块和包，有以下一些优势：

1. 简单化，将复杂的问题拆解成一个个简单的子问题。
2. 可维护性，对于动辄几百上千行的代码，我们更加乐于看到一个几十行的函数，或者逻辑边界清晰的模块或包，分解后，可以将依赖降低，修改一个函数或者模块而不用担心“牵一发而动全身”的问题。
3. 可重用性，具有独立功能的模块和包可以很方便的进行分发，以及作为别的项目的依赖，进行代码重用。
4. 作用域，模块和包提供了独立的命名空间，可以防止变量、对象、函数名重复。

我觉得，模块和包是给人类看的，就像域名解析系统一样，对于机器而言，这些都不是必须的。

---

## 模块

一般有三种方式可以在 Python 中定义模块：

1. 由 Python 语言编写
2. 由 C 语言编写，并在 run-time 时动态加载
3. 内建模块（built-in module），内置在了 Python 解释器中。

第二种一般出现在需要要求性能的外部库，第三种在安装 Python 环境时，已经默认包含在了解释器中。

### 如何编写模块

这里考虑第一种——由 Python语言编写的模块，很简单，一个以 py 为扩展名的文件（比如：mod.py）就是一个模块。

不需要特殊的语法，就像普通的脚本一样，你可以在模块中定义变量，函数，对象：

mod.py
```python
s = "If Comrade Napoleon says it, it must be right."
a = [100, 200, 300]

def foo(arg):
    print(f'arg = {arg}')

class Foo:
    pass
```

### 如何使用模块

模块在使用之前，必须进行导入，大部分语言都是用 import 作为关键字，Python 也一样：

main.py
```python
import mod

print(mod.s)
print(mod.a)
mod.foo(1)
c = mod.Foo()
print(c)
```

### 模块的搜索路径

刚刚建立了两个代码文件：mod.py 和 main.py，并且在 main.py 中导入了 mod 这个模块，问题在于 Python 解释器是如何根据一句简单的 `import mod` 就定位到了 mod.py 文件的？难道只会搜索当前目录么？如果我们把 mod.py 放到别的目录下，Python 解释器是否还能搜到这个模块？

实际上，在执行导入的时候，Python 解释器会按照一个预定义的 list 来进行搜索，这个预定义的路径在 sys 模块中可以找到：

Windows
```python
>>> import sys
>>> sys.path
['', 'C:\\Users\\ABC\\ProgramLanguage\\Python\\Python311\\python311.zip', 'C:\\Users\\ABC\\ProgramLanguage\\Python\\Python311\\DLLs', 'C:\\Users\\ABC\\ProgramLanguage\\Python\\Python311\\Lib', 'C:\\Users\\ABC\\ProgramLanguage\\Python\\Python311', 'C:\\Users\\ABC\\ProgramLanguage\\Python\\Python311\\Lib\\site-packages']
```

Debian
```python
>>> import sys
>>> sys.path
['', '/usr/lib/python311.zip', '/usr/lib/python3.11', '/usr/lib/python3.11/lib-dynload', '/usr/local/lib/python3.11/dist-packages', '/usr/lib/python3/dist-packages']
```

sys.path 就是 Python 解释器在执行 import 模块时，搜索的路径，该 list 主要由以下几部分组成：

1. 当前工作目录（如果以交互方式运行）或者脚本的所属目录（以脚本方式运行）
2. PYTHONPATH 环境变量
3. 安装 Python 时配置的目录

无论是 Windows 还是 Linux，目前在没有指定环境变量的情况下，都是由当前工作目录和 Python 安装时的目录组成：

Windows：
- '' 空字符串：这代表当前工作目录（Current Working Directory），也就是你运行Python脚本时所在的目录。
- python311.zip：包含Python标准库的ZIP压缩文件，为了更方便地分发和减少磁盘占用，Python可以将一部分标准库文件打包成一个单独的 .zip 文件。解释器在导入模块时能够直接读取这个压缩包内的文件，就像它是一个普通的文件夹一样。
- DLL：这是存放Windows动态链接库（DLLs） 的目录，某些Python模块，特别是那些与系统底层交互或为了提升性能而用C语言编写的模块，在Windows平台上依赖于 .dll 文件。这个目录就是专门为这些模块提供所需的运行时依赖。
- Lib：这是Python标准库（Standard Library） 的主目录，包含了 Python 自带的大量模块，比如 os、re、json 等。
- Python311：Python 安装根目录。
- site-packages：第三方库（Third-party Packages） 的安装目录，用 pip install xxx 安装的包默认都会放在这里，比如 flask、numpy、pandas。

Linux：
- lib-dynload：这个目录包含了用 C 语言编写、为了提升性能而编译成的共享对象文件（.so 文件，相当于 Windows 的 .pyd 或 .dll 文件）。
- /usr/local/lib/python3.11/dist-packages：系统级第三方库 的安装目录（通过 pip 等工具安装）。
- /usr/lib/python3/dist-packages：由系统包管理器（如 apt）安装的 Python 包的目录。

这里可以通过以下几种方式修改 sys.path

1. 制定环境变量 PYTHONPATH
2. 使用虚拟环境
3. 手动修改 sys.path

第一种，指定环境变量 PYTHONPATH，以设置一个自定义目录为例：

powershell
```
设置单个
$env:PYTHONPATH='C:\mylib'
设置多个
$env:PYTHONPATH="C:\my\extra\modules;D:\another\path"
```

cmd
```
设置单个
set PYTHONPATH=C:\mylib
设置多个
set PYTHONPATH=C:\my\extra\modules;D:\another\path
```

bash
```
设置单个
export PYTHONPATH=/home/abc/mylib
设置多个
export PYTHONPATH=/home/abc/my/modules:/opt/another/path
```

注意：Windows 下路径分隔符是 ;，Linux/macOS 是 :。

设置后，Python 启动时会优先在这些目录里找模块（会把这些路径加在 sys.path 的开头）。

第二种，建立虚拟环境，虚拟环境会改变 site-packages，其他部分保持不变：

```
>>> import sys
>>> sys.path
['', 'C:\\Users\\ABC\\ProgramLanguage\\Python\\Python311\\python311.zip', 'C:\\Users\\ABC\\ProgramLanguage\\Python\\Python311\\DLLs', 'C:\\Users\\ABC\\ProgramLanguage\\Python\\Python311\\Lib', 'C:\\Users\\ABC\\ProgramLanguage\\Python\\Python311', 'C:\\Users\\ABC\\Desktop\\test\\venv', 'C:\\Users\\ABC\\Desktop\\test\\venv\\Lib\\site-packages']
```

第三种，手动在 sys.path 使用 append

### 确定模块的文件路径

Python 提供了一个内置属性 `__file__` 来查询模块的文件路径

```
>>> import mod, sys, json, re, time
>>> mod.__file__
'C:\\Users\\ABC\\Desktop\\test\\mod.py'
>>> sys.__file__
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: module 'sys' has no attribute '__file__'. Did you mean: '__name__'?
>>> json.__file__
'C:\\Users\\ABC\\ProgramLanguage\\Python\\Python311\\Lib\\json\\__init__.py'
>>> re.__file__
'C:\\Users\\ABC\\ProgramLanguage\\Python\\Python311\\Lib\\re\\__init__.py'
>>> time.__file__
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: module 'time' has no attribute '__file__'. Did you mean: '__name__'?
>>>
```

对于纯 Python 模块/包（.py 文件或包目录）,Python 加载时会保存它的源代码位置，所以有 `__file__` 属性。

而对于内置模块（built-in modules），是 Python 解释器用 C 实现并直接编译进来的，没有对应的 .py 文件，也就没有 `__file__` 属性。

---

## import 的语法

### 导入单个模块
import 的语法有很多变型，最简单的一种是前文提及的：

```
import <module_name>
```

例如：

```
import mod
```

这个语句会把 mod 作为被调用的模块，放到调用者的符号表中，此时在 mod 中定义的那些变量，函数，方法并未直接导入到调用者的符号表，也就是说访问 mod 是可以的，但是直接访问 mod 中的元素是会报错的，想要访问 mod 中的元素，使用 . 符号：

```python
import mod

print(mod)

print(mod.s)
```

### 导入多个模块

导入多个模块，只需要把模块用逗号隔开或者分成多个 import 语句：

```python
import mod1, mod2, mod3

# or

import mod1
import mod2
import mod3
```

### 改名

考虑以下场景：

当前的 main.py 已经存在一个变量——mod，然后我们又需要导入 mod 模块

```python
import mod

mod = 1

print(mod)
```

这里，main 中的符号 mod 与模块 mod 发生了冲突，Python 解释器对于这种情况不会报错，仅仅会执行简单的覆盖（后面的覆盖前面的）。

对于这种场景，我们可以使用以下语法：

```
import <module_name> as <alt_name>
```

上例可以改为：

```python
import mod as my_mod

mod = 1

print(mod)
print(my_mod)
```

### 从模块中导入元素

前面已经描述了如何使用 import 把模块本身导入到当前符号表中，并且使用 `<module>.<element>` 的形式访问模块中的元素，我们也可以通过以下语法把元素直接导入到当前符号表，就可以直接访问该元素。

```
from <module_name> import <name(s)>
```

例如：

```python
from mod import s

print(s)
```

也可以导入多个，用逗号隔开：

```python
from mod import s, a, foo

print(s)
print(a)
print(foo(1))
```

如果导入的元素和当前符号表有重名，可以使用 as 进行修改：

```python
from mod import s as mod_s, a as mod_a, foo

s = 123
a = 456

print(s)
print(mod_s)

print(a)
print(mod_a)

print(foo(s))
```

### 从模块中导入所有元素

在 Python 的 import 机制中，有一个看起来很富有诱惑力的写法：

```
from <module_name> import *
```

这样写的效果是，把模块中除了下划线开头的元素全部导入到当前的符号表中。

在官方的风格指南中或者别的生产实践指南中，明确规定了这种写法不应该出现在生产环境中，主要有以下几个理由

第一，这种写法会导致命名空间污染，当前的符号表的部分元素可能会被覆盖。

假设当前符号表有一个 cnt 变量，模块中也有一个 cnt 变量，就会发生覆盖，尽管你可能根本不想把模块中的 cnt 导入进来。

第二， 代码的可读性很差。

如果每一个模块都这么写：

```python
from mod1 import *
from mod2 import *

print(s)
```

你无法知道 s 到底属于 mod1 还是 mod2，或者，两个模块中都有 s


所以，在生产环境中，极为不推荐该写法。

### import 相关的异常

与 import 相关的一场主要有两个：ModuleNotFoundError 和 ImportError。

ModuleNotFoundError 发生在 import 一个不存在的包，准确的来说，是在 sys.path 中找不到的时候触发。

ImportError 发生在模块找到了，但是从模块中导入的元素找不到。

```
>>> import xxx
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ModuleNotFoundError: No module named 'xxx'

>>> from math import xxx
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ImportError: cannot import name 'xxx' from 'math' (unknown location)
```

---

## 包

在开发简单脚本的情况下，一个py文件足矣，稍稍复杂一些的项目，也许会编写多个模块，然后在 main 中调用多个模块的变量、函数或者对象。那么更进一步，更大型，更复杂的项目可能会有几十上百个模块。单一扁平化的结构就不适用了，同样的，和文件系统类似，Python 引入了包（Package）的概念，对模块进行分层管理。

和模块一样，包的概念提供了命名空间（namespace），这种命名空间以点（dot）分隔，类似域名系统。

创建包很直观，如果说模块等同于文件，那么包就是目录。

### 导入包

假设目录结构如下：

```
-pkg
  |-mod1.py
  |-mod2.py
```
文件内容如下：

```
mod1.py:

def foo():
    print('[mod1] foo()')

class Foo:
    pass


mod2.py:

def bar():
    print('[mod2] bar()')

class Bar:
    pass

```

如果 pkg 目录位于 sys.path 列表中某一个目录下可以找到的位置，那么就可以使用点符号 import 到代码中：

```python
import pkg.mod1, pkg.mod2

print(pkg.mod1())

x = pkg.mod2.Bar()
print(x)
```

上面的写法是导入了模块，也可以导入模块中的元素：

```python
from pkg.mod1 import Foo as F

foo = F()

print(foo)
```

### 和导入模块不同的点

当我们导入一个模块，那么通过点符号就可以获得模块下的元素，但是当我们导入一个包，却并不能通过点符号导入包下的元素。

```python

import pkg

print(pkg)

print(pkg.mod1)
```

尝试获取 pkg 下的元素 mod1 会爆出错误：

```
AttributeError: module 'pkg' has no attribute 'mod1'
```

所以，单纯的导入一个包，并没有太多用处。

### 包的初始化

如果我们在 pkg 包下面，放置一个`__init__.py`文件，那么该文件代码会在 pkg 包被导入的时候被调用，一般会在该文件中放置一些初始化代码。

例如，目录结构如下：

```
-pkg
  |-__init__.py
  |-mod1.py
  |-mod2.py

```

文件内容如下:

```python
print(f'Invoking __init__.py for {__name__}')
A = ['quux', 'corge', 'grault']
```

包一旦被导入，`__init__.py`代码就会被执行：

```
>>> import pkg
Invoking __init__.py for pkg
>>> pkg.A
['quux', 'corge', 'grault']

```

除了初始化的功能外，一般会在`__init__.py`放置一些包下的全局变量，比如这个 A 列表，可以被 mod1 和 mod2 导入。

mod1.py:

```python

def foo():
    from pkg import A
    print('[mod1] foo() / A = ', A)

class Foo:
    pass

在上一小节提到，导入 pkg 并不能调用 pkg.mod1，但是如果在`__init__.py`中提前导入，就可以了：


```python
print(f'Invoking __init__.py for {__name__}')
import pkg.mod1, pkg.mod2
```

### 包下的 init 文件

很多教程都说，在 Python 中，要把一个目录作为一个包，那么该目录下一定要有个`__init__.py`文件，但这是很早之前的事情了，在 Python3.3 之后，引入了 Namespace Package 的概念，不需要`__init__.py`文件，也可以把该目录作为 Python 包。

### import * 在包中的使用

如果我们在 module 中使用了`import *`，那么模块下的所有非下划线开头的元素都会被导入到当前命名空间中，这是个不好的实践方式，那如果在 package 使用`import *` 呢？

实际上，在包内导入所有元素是没有任何效果的：

```
>>> dir()
['__annotations__', '__builtins__', '__cached__', '__doc__', '__file__', '__loader__', '__name__', '__package__', '__spec__']
>>> from pkg import *
>>> dir()
['__annotations__', '__builtins__', '__cached__', '__doc__', '__file__', '__loader__', '__name__', '__package__', '__spec__']
```

可以看到，前后两次 dir 没有任何变化，Python 的 module 和 package 在`import *`上表现的不一致，我猜测主要是 package 不可控，一个 package 可能包含大量的嵌套和初始化代码逻辑，可能会对性能造成很大的问题。

想要使得 package 也可以`import *`，python 提供了一个约定，在`__init__.py`中定义一个列表元素：`__all__`，该列表包含了 package 下需要导入的子模块。

假设 pkg 下有两个子模块：mod1.py 和 mod2.py，在`__init__.py`定义：

```python
# pkg/__init__.py

__all__ = ['mod1',]

```
那么在使用`from pkg import *`的时候，则只会导入 mod1 这个子模块。

```
>>> dir()
['__annotations__', '__builtins__', '__cached__', '__doc__', '__file__', '__loader__', '__name__', '__package__', '__spec__']
>>> from pkg import *
>>> dir()
['__annotations__', '__builtins__', '__cached__', '__doc__', '__file__', '__loader__', '__name__', '__package__', '__spec__', 'mod1']
>>> mod1.foo()
foo
>>> 
```

### 子包

类似域名系统和文件系统，python 的包是可以包含多个层级的。


---



## 关于模块和包的 import * 总结

上面没有提到的是，`__all__`如果定义在 module 中也是有着约束导入的效果的，和 package 是一样的，所以如果不想要`from mod1 import *`导入所有元素，那么可以在 mod1 模块中定义`__all__`。

总的来说：

- 对于包而言，如果没有定义`__all__`，则`import *`什么子模块也不会导入。
- 对于模块而言，如果没有定义`__all__`，则`import *`会导入除了下划线开头的所有元素。



---

## 参考

1. https://realpython.com/python-modules-packages/#python-modules-overview
2. https://docs.python.org/3/tutorial/modules.html
3. https://docs.python.org/3/library/sys\_path\_init.html#sys-path-init
