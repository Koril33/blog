---
title: "Python的异常机制"
date: 2025-07-31T10:10:00+08:00
summary: "Python 如何处理，抛出异常？以及异常的体系和机制"
---

## 目录

[TOC]

---

## 前言

主流编程语言主要有两种处理异常的模型，一种是错误返回值，比如：Go 和 C，都是通过返回值来判断是否发生异常情况。另外一种，是异常模型，比如：Java，Python，C#，C++等。

本文主要讲述 Python 的异常机制。

---

## SyntaxError

有一种异常比较特殊，它不是发生在运行时（RunTime），而是在 Python 解释器解析代码时可能会爆出的异常。

例如：

```sh

>>> print())
  File "<stdin>", line 1
    print())
           ^
SyntaxError: unmatched ')'

>>> print(')
  File "<stdin>", line 1
    print(')
          ^
SyntaxError: unterminated string literal (detected at line 1)

>>> print(;)
  File "<stdin>", line 1
    print(;)
          ^
SyntaxError: invalid syntax

>>> try:
...   print('ok')
...
  File "<stdin>", line 3

    ^
SyntaxError: expected 'except' or 'finally' block

```

Python 是一种解释型语言，解释器在执行代码之前会先对代码进行语法检查，如果代码语法违法了 Python 语言的规则，就会抛出 SyntaxError。

---

## Exception

如果代码语法没问题，程序运行起来后，也可能会存在各种运行时的错误，一般把这种错误都称为异常（Exception）。

以最经典的除零异常为例：

```python
print(1/0)
```

异常信息如下：

```
Traceback (most recent call last):
  File "C:\Users\ABC\Desktop\main.py", line 1, in <module>
    print(1/0)
          ~^~
ZeroDivisionError: division by zero
```

错误信息的最后一行说明程序遇到了什么类型的错误。异常有不同的类型，比如这里的 ZeroDivisionError。

可以简单的通过类型判断错误大致上属于类别，对于所有内置异常都会打印出异常类型。

异常类型后面的内容则是更加详细的描述信息。

错误信息的开头使用堆栈回溯的形式展示发生异常时的上下文环境，比如存在多个函数嵌套的异常：

```python
def a():
    b()

def b():
    c()

def c():
    print(1/0)

if __name__ == '__main__':
    a()
```

异常信息如下：

```
Traceback (most recent call last):
  File "C:\Users\ABC\Desktop\main.py", line 11, in <module>
    a()
  File "C:\Users\ABC\Desktop\main.py", line 2, in a
    b()
  File "C:\Users\ABC\Desktop\main.py", line 5, in b
    c()
  File "C:\Users\ABC\Desktop\main.py", line 8, in c
    print(1/0)
          ~^~
ZeroDivisionError: division by zero
```

---

## 抛出异常

上面提到的是程序自动抛出异常，但我们有时候碰到一些特殊情况，想要手动触发异常，那么可以使用 raise 关键字。

假设现在需要用户输入一个数字，需要比 10 大，否则抛出一个异常

```
num_str = input('Enter a number: ')

num = int(num_str)

if num > 10:
    print('ok')
else:
    raise Exception(f'number: {num} <= 10')
```

输入一个小于或等于 10 的数字，抛出异常信息：

```
Enter a number: 1
Traceback (most recent call last):
  File "C:\Users\ABC\Desktop\main.py", line 8, in <module>
    raise Exception(f'number: {num} <= 10')
Exception: number: 1 <= 10
```
---

## 捕获以及处理异常

如果对于程序的异常没有做任何处理，那么，Python 会自动调用 sys.exit() 来终止程序的执行。

但很多时候，我们碰到异常情况也许可以做一些补救或者记录性的事情，这时候就需要在程序的某个地方捕获异常。

还是以用户输入一个数字为例子：

```python
num_str = input('Enter a number: ')

num = int(num_str)

print(f'user input number: {num}')
```
如果用户输入错误，比如输入了字符串，int 函数会抛出异常，而我们没有任何处理，程序打印完异常信息后，直接退出；

```
Enter a number: f1
Traceback (most recent call last):
  File "C:\Users\ABC\Desktop\main.py", line 3, in <module>
    num = int(num_str)
          ^^^^^^^^^^^^
ValueError: invalid literal for int() with base 10: 'f1'
```
这时候可以使用 try-except 捕获和处理异常：

```
num_str = input('Enter a number: ')

try:
    num = int(num_str)
except:
    print(f'number incorrect ({num_str=}), use default number.')
    num = -1

print(f'user input number: {num}')
```
这里使用 try 捕获了 int 可能由于用户输入错误而抛出的异常，并且在 except 中进行了处理（打印一条信息并使用默认值），由于我们处理了异常，所以程序并不会直接退出。

try 语句的目的在于给一组语句指定异常处理器和清理性质的代码。

except 子句跟随 try 语句出现，主要作用是匹配 try 块中代码可能出现的异常类型，如果 except 后面没有任何表达式，那么该 except 子句将匹配任何类型的异常。

在 except 后面不加任何异常类型，初看感觉很简洁，写起来也方便，但在生产中是一种比较偷懒的写法。

假设 try 块中有多行代码，都可能引发各种类型的异常：

```
import requests

try:
	response = requests.get('https://example.com', timeout=5)
	f = open('./content.txt', 'w', encoding='utf-8')
	f.write(response.text)
except:
	print('Some errors happened, but I do not know...')
```

代码逻辑很简单，通过网络请求一个站点，然后把 HTTP 响应的 text body 写到本地的一个文件中，这个过程可能会有各种各样的异常，服务器请求错误（404，500），网络超时，文件无法打开，文件写入错误等等。

如果 except 简单忽略掉后面的异常类型，那么它会匹配到所有异常类型，最终我们在 except 的处理代码中，只能知道出错了，但是不知道具体发生了什么错误。

最常见的一个请求错误是读取超时，国内访问外网的时候经常发生，除此之外 HTTP response code 不为 200 的情况也时有发生，如果我们需要将这些异常分开捕获，那么需要在每一个 except 子句中，指定对应的异常类型：

```python
import requests

try:
    response = requests.get("http://xxx123example.com", timeout=10)
    response.raise_for_status()
    print(response.text)
except requests.exceptions.ReadTimeout as e:
    print(f'Read timeout error: {e}')
except requests.exceptions.ConnectionError as e:
    print(f'Connection error: {e}')
except requests.exceptions.HTTPError as e:
    print(f'HTTP error: {e}')

print('task end...')
```

不同的异常类型被不同的 except 子句捕获后处理。

---

## 捕获的顺序

try-except 语句执行的顺序如下：

1. 执行 try 块中的多行代码。
2. 如果 try 块中多行代码没有抛出任何异常，则跳过 except 子句，try-except 执行完毕。
3. 如果在 try 块中某一行代码触发了异常，那么跳过 try 块中剩余的其他代码，进入到 except 匹配。
4. except 匹配自上到下，如果异常的类型和某一个 except 子句后指定的异常相匹配，则进入到 except 块执行代码，try-except 执行完毕。
5. 如果 except 没有匹配到，则该异常会被传递到外层，如果一直没有找到合适的处理器，那么它就是一个未处理异常，程序将停止运行并输出一条错误消息。

try-except 中可以存在多个 except 子句，但最多只有一个 except 会被执行。

从上到下的捕获顺序规则：

1. try 块出现异常的时候，Python 会从上到下按顺序，依次检查每一个 except，尝试匹配。
2. 第一个匹配成功的 except 会被执行，其余忽略。
3. 存在多个 except 时，要保证子类异常在前，父类异常在后，否则会导致，父类先捕获异常而子类永远不会执行。

最后一个规则是由于 exception 存在继承关系，Python 内部异常继承关系如下：

```
BaseException
 ├── BaseExceptionGroup
 ├── GeneratorExit
 ├── KeyboardInterrupt
 ├── SystemExit
 └── Exception
      ├── ArithmeticError
      │    ├── FloatingPointError
      │    ├── OverflowError
      │    └── ZeroDivisionError
      ├── AssertionError
      ├── AttributeError
      ├── BufferError
      ├── EOFError
      ├── ExceptionGroup [BaseExceptionGroup]
      ├── ImportError
      │    └── ModuleNotFoundError
      ├── LookupError
      │    ├── IndexError
      │    └── KeyError
      ├── MemoryError
      ├── NameError
      │    └── UnboundLocalError
      ├── OSError
      │    ├── BlockingIOError
      │    ├── ChildProcessError
      │    ├── ConnectionError
      │    │    ├── BrokenPipeError
      │    │    ├── ConnectionAbortedError
      │    │    ├── ConnectionRefusedError
      │    │    └── ConnectionResetError
      │    ├── FileExistsError
      │    ├── FileNotFoundError
      │    ├── InterruptedError
      │    ├── IsADirectoryError
      │    ├── NotADirectoryError
      │    ├── PermissionError
      │    ├── ProcessLookupError
      │    └── TimeoutError
      ├── ReferenceError
      ├── RuntimeError
      │    ├── NotImplementedError
      │    ├── PythonFinalizationError
      │    └── RecursionError
      ├── StopAsyncIteration
      ├── StopIteration
      ├── SyntaxError
      │    └── IndentationError
      │         └── TabError
      ├── SystemError
      ├── TypeError
      ├── ValueError
      │    └── UnicodeError
      │         ├── UnicodeDecodeError
      │         ├── UnicodeEncodeError
      │         └── UnicodeTranslateError
      └── Warning
           ├── BytesWarning
           ├── DeprecationWarning
           ├── EncodingWarning
           ├── FutureWarning
           ├── ImportWarning
           ├── PendingDeprecationWarning
           ├── ResourceWarning
           ├── RuntimeWarning
           ├── SyntaxWarning
           ├── UnicodeWarning
           └── UserWarning
```
考虑一下程序：

```python
import requests

try:
    response = requests.get("http://123example.com", timeout=1)
    response.raise_for_status()
    print(response.text)
except IOError as e:
    print(f'io error: {e}')
except requests.exceptions.ReadTimeout as e:
    print(f'Read timeout error: {e}')
except requests.exceptions.ConnectionError as e:
    print(f'Connection error: {e}')
except requests.exceptions.HTTPError as e:
    print(f'HTTP error: {e}')

print('task end...')
```

由于把 IOError 放在了最上面，而下面所有异常都直接或者间接继承自 IOError，导致发生 ReadTimeout 或者 HTTPError 都会直接被 except IOError 这里直接捕获，后面的 except 就没有了意义。

所以一般建议是：当存在多个 except 块时，从具体的子类再到宽泛的父类异常。

---

## else

如果控制流离开 try 子句体时没有引发异常，并且 try 中没有执行 return, continue 或 break 语句，可选的 else 子句将被执行。

else 语句中的异常不会由之前的 except 子句处理。

---

## finally

finally 子句的目的在于资源清理，无论 try 中有没有发生异常，finally 都会执行，而且和 else 子句不同的一点是，如果 try 中有 return，else 不会执行，而 finally 子句会执行。

```python
def foo():
    try:
        a = 1
        print('try')
        return a
    except:
        pass
    else:
        print('else')
        return a * 2
    finally:
        print('finally')
        return a * 3

res = foo()
print(res)
```

上面代码的结果是：

```
try
finally
3
```
由于 try 中有 return，else 子句不可达，但是 finally 子句一定会执行，并且最后的返回值是 3，try 的 return a 被 finally 中的 return a * 3 覆盖了。


finally 最有用的地方在于处理文件句柄，磁盘、网络IO的资源清理和关闭，如果这些代码不放在 finally 中，那么一旦碰到未处理的异常，这些清理性质的代码可能永远也不会执行，这会造成句柄，连接，内存的泄露。

---

## 异常链

理想情况下，try 中发生异常，except 去处理异常，程序走到后续的流程。

但现实是，except 处理异常的代码也可能产生新的异常！

这里就涉及到了异常链的概念，它指的是一个新的未处理异常发生在了 except 内部，Python 会将已被处理的异常信息附加在它上面。

```python
def a():
    print("a")
    raise Exception('oops...')

def b():
    try:
        a()
    except Exception as e:
        print("b")
        raise ValueError('value err b') from e

b()
```
输出结果：

```
a
b
Traceback (most recent call last):
  File "C:\Project\Study\study-python-exception\main.py", line 11, in b
    a()
  File "C:\Project\Study\study-python-exception\main.py", line 7, in a
    raise Exception('oops...')
Exception: oops...

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "C:\Project\Study\study-python-exception\main.py", line 16, in <module>
    b()
  File "C:\Project\Study\study-python-exception\main.py", line 14, in b
    raise ValueError('value err b')
ValueError: value err b
```

ValueError 是 b 函数在处理 a 函数抛出的异常时，新产生的异常，如果要指明 ValueError 就是由 Exception 引发的，则可以在 raise 后面添加 from：

```python
raise ValueError('value err b') from e
```

输出如下：

```
a
b
Traceback (most recent call last):
  File "C:\Project\Study\study-python-exception\main.py", line 11, in b
    a()
  File "C:\Project\Study\study-python-exception\main.py", line 7, in a
    raise Exception('oops...')
Exception: oops...

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "C:\Project\Study\study-python-exception\main.py", line 16, in <module>
    b()
  File "C:\Project\Study\study-python-exception\main.py", line 14, in b
    raise ValueError('value err b') from e
ValueError: value err b
```

---

## 参考

1. https://docs.python.org/zh-cn/3.13/library/exceptions.html#bltin-exceptions
2. https://docs.python.org/zh-cn/3.13/tutorial/errors.html
3. https://realpython.com/python-exceptions/
4. https://docs.python.org/zh-cn/3.13/reference/executionmodel.html#exceptions
5. https://docs.python.org/zh-cn/3.13/reference/compound_stmts.html#try
