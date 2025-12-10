---
title: "上下文管理器"
date: 2025-12-09T16:35:05
summary: "解决重复性的样板代码和防止资源泄露的有效工具"
---

## 前言

在持久运行的服务端系统，由编程错误导致的一个很常见的问题是资源泄露。

资源可以指代很多东西（内存、CPU、磁盘、文件句柄、socket连接，数据库连接，锁等等），但它们都有一个共同特点，那就是有限性，在计算机里的资源几乎都存在硬件层面或者操作系统层面的限制。

本文所介绍的上下文管理器主要处理两种场景：

第一种场景就是管理资源，管理这些有限的资源是一个及其耗时费心的工作，往往我们随取随用，但是忘记编写清理资源的相关代码是一种常见的编程错误。

另外还有一种场景，在处理一些业务逻辑涉及重复性的非功能需求，比如：日志记录，链路追踪，性能检测等等。这些样板代码往往是重复的，它们的模式是在业务代码前执行一段相同的代码，在业务代码执行完毕或者发生异常再执行另一段相同的代码。

以上提到的两种场景，在 Python 中都可以通过 Context Manager（上下文管理器）进行封装处理。

---

## 场景一：资源泄露

资源泄露往往是编码过程的疏忽导致，我们编写了对资源取用的代码后，没有编写及时清理关闭的代码，导致长期运行的软件（比如：web服务端程序），不断地创建资源对象，却没有进行释放。

一个典型的案例是文件的资源泄露：https://realpython.com/why-close-file-python/

处理资源开启和关闭的方案有两种：

1. try...finally
2. with

第一种`try...finally`属于通用方案，适用于所有资源对象，因为在 finally 块的语句一定会执行（除了机器被直接断电或者 Python 进程非正常退出），所以几乎所有涉及资源管理的示例代码都采用了`try...finally`语句。

`try...finally`的优点是通用性，不需要资源对象实现什么协议，缺点是冗长，以及存在忘记在 finally 块中调用资源清理代码的可能性。

文件：

```python
file = open('student.txt', 'w')
try:
    file.write('bob\nalice')
finally:
    file.close()
```

socket：

```python
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
try:
    s.connect(("example.com", 80))
    s.send(b"GET / HTTP/1.0\r\n\r\n")
    while True:
        data = s.recv(1024)
        if not data:
            break
        print(data)
finally:
    s.close()
```


在 try 块中无论遇到 return，break/continue，Exception，最终 finally 的 close() 都会执行。

第二种的 with statement 就是本文的主角，它的优点是代码简洁，不会忘记执行 close 之类的清理代码（因为使用的时候，根本不用写，也就不会忘记了），缺点是需要对象实现一些固定的协议，不如`try...finally`那么通用。

假设一个项目有 10 个地方需要用到资源处理，那么意味着要写 10 次 `try...finally`代码，每增加一次处理，都会增加泄露的风险。所以不如把释放和清理资源的代码提前写在某个地方（实现协议），后续的调用不用再考虑重复编写资源处理的代码。

file:

```python
with open('student.txt', 'w') as file:
    file.write('bob\nalice')
```

socket:

```python
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.connect(("example.com", 80))
    s.send(b"GET / HTTP/1.0\r\n\r\n")
    while True:
        data = s.recv(1024)
        if not data:
            break
        print(data)
```

with statement 将资源相关的代码进行了抽离和隐藏，这样程序员无需担心忘记在 finally 中编写资源释放的代码，with 中只需要编写业务逻辑，整体看起来更加简洁。

---

## 场景二：样板代码

在编程中，成对操作是很常见的，上节提到的资源管理就是成对操作的一种，假设 M/N 是成对操作，那么编程语句通常长这样子：

```python
# 在业务代码前执行 M
M()
# 一些业务代码
some_task()
# 在业务代码后执行 N
N()
```

为了防止 N 由于业务代码抛异常导致无法执行，通常将 N 放在 finally 中：

```python
# 在业务代码前执行 M
M()
# 一些业务代码，可能会抛出异常
try:
    some_task()
finally:
    # 在业务代码后执行 N
    N()

```

资源处理（获取和释放）的成对操作只是其中一种，还有很多类似的操作：

- startup/cleanup
- init/destroy
- open/close
- connect/disconnect
- acquire/release
- setup/teardown
- change/reset
- begin transaction/commit or rollback
- start/stop
- bootstrap/shutdown
- bind/unbind
- mount/unmount

凡是涉及到上述的成对操作，几乎都避免不了相似的样板代码，所以 Python 的 [PEP343](https://peps.python.org/pep-0343/) 将这种样板代码抽象成了 with 语法。

几乎所有主流语言都提供了这种语法糖：

1. C++/Rust 的 SBRM、RAII，离开作用域就会自动释放资源
2. C# 的 using，需要资源对象实现 IDisposable 接口
3. Java 的 try-with-resource，需要资源对象实现 AutoCloseable 接口
4. Python 的 with statement，需要实现`__enter`和`__exit__`方法
5. Kotlin 的 use 函数

当大量的样板代码和业务代码混合起来，整体就显得很乱，而且每个地方都要重复编写，把这些通用的`try...finally`抽象出来，放在一个地方，整体就会清晰很多，我个人感觉，这种思想类似 AOP（Aspect-oriented programming）。

---

## 实现

许多官方库的资源对象都实现了上下文管理协议，所以可以直接使用 with statement，如果需要自定义上下文管理器，就需要自己实现上下文管理协议。

有两种方式可以自定义上下文管理器：

1. 实现协议的类
2. 使用 @contextmanager 装饰的函数

### 类

自定义的上下文管理器实际上就是一个实现了上下文管理协议的类。上下文管理协议要求类包含两个特殊的方法：

```

object.__enter__(self)

__enter__ 定义了进入上下文环境时要准备的事情，比如：打开文件、获取锁、开始事务、创建某些对象。
__enter__ 的返回值很重要，返回值代表了上下文管理器，一般有两类返回值：返回自身或者相关对象。

object.__exit__(self, exc_type, exc_value, traceback)

__exit__ 定义了退出上下文环境要做的事情，比如：关闭文件，释放锁，提交或者回滚事务，清除某些对象。
__exit__ 的三个入参和异常处理有关，如果退出上下文环境时没有引发异常，那么三个参数都是 None，如果引发了异常，但是想要抑制该异常，那么需要 __exit__ 返回 True。

```

with statement 具体的执行步骤如下：

1. 通过 with 语句中的表达式求值得到上下文管理器对象
2. 加载上下文管理器对象的`__enter__`方法，供后续调用
3. 加载上下文管理器对象的`__exit__`方法，供后续调用
4. 调用`__enter__`方法
5. 如果 with 表达式后面跟着 as 语句，as 指定的目标可以是单个名称，也可以是解包组合，会把`__enter__`的返回对象赋给 as 后面的目标
6. 执行 with 中的主体代码块
7. 调用`__exit__`方法
8. 如果在执行 with 主体代码块中发生异常，则把异常信息传递给`__exit__`，否则传递 (None, None, None)，并且`__exit__`返回 True 表示抑制异常，返回 False 则会重新向外抛出原异常

下面给出 with 语句翻译后的同等代码：

```python
with EXPRESSION as TARGET:
    SUITE
```

等价于（对应上面的每一个步骤）

```python
manager = (EXPRESSION)
enter = type(manager).__enter__
exit = type(manager).__exit__
value = enter(manager)
hit_except = False

try:
    TARGET = value
    SUITE
except:
    hit_except = True
    if not exit(manager, *sys.exc_info()):
        raise
finally:
    if not hit_except:
        exit(manager, None, None, None)
```

with 是一种 try...finally 样板代码的语法糖，它保证了成对函数中，如果`__enter__`成功执行（意味着进入了上下文环境），那么`__exit__`一定会执行。

下面以一个简单的案例来看下自定义上下文管理器如何编写。

对于测量一段代码的耗时往往会产生以下重复的样板代码：

```python
# 记录开始时间
start = int(time.time() * 1000)
try:
    # 一些业务逻辑
    some_task()
finally:
    # 记录结束时间
    end = int(time.time() * 1000)
    print(f'Task cost: {end-start} ms.')
```

这里的成对函数就是在进入业务逻辑代码前，需要记录一次开始的毫秒戳，退出业务逻辑代码后，再记录一次毫秒戳。

抽离成上下文管理器：

```python
class TaskTimer:

    def __init__(self, name):
        self.name = name

    def __enter__(self):
        self.start = int(time.time() * 1000)
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.stop = int(time.time() * 1000)
        print(f'Task<{self.name}> cost: {self.stop - self.start} ms')

```

调用时，这些样板代码就无需重复书写了：

```python

with TaskTimer(name='simple task'):
    print('some task start')
    time.sleep(2)
    print('some task end')
```


### 装饰器

暂略

---

## 参考

1. https://peps.python.org/pep-0343/
2. https://realpython.com/python-with-statement/
3. https://docs.python.org/3/library/contextlib.html
4. https://docs.python.org/3/reference/datamodel.html#with-statement-context-managers
5. https://docs.python.org/3/reference/compound_stmts.html#with