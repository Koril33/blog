---
title: "PEP282-日志系统-译文"
date: 2025-07-05T14:00:00+08:00
summary: "PEP 282 – A Logging System 的中文版"
---

## 原文

https://peps.python.org/pep-0282/

该 PEP 主要描述日志系统的基本概念和标准。

---

## PEP 282 – 日志系统

作者: Vinay Sajip, Trent Mick
状态: Final
类型: Standards Track
创建时间: 04-Feb-2002
Python 版本呢: 2.3

### 摘要

此 PEP 描述了 Python 标准库的日志记录包。

基本上该系统涉及到用户创建一个或者多个 logger 对象，可以使用该对象的特定方法记录调试日志（debugging notes），一般信息（general information），警告（warnings），错误（errors）等等。不同的日志级别（logging levels）用以区分那些重要的信息和不太重要的信息。

维护一个以名字作为单例 logger 对象的注册表，有利于：

1. 存在不同的逻辑日志流（或“通道”）（例如，一个用于“zope.zodb”内容，另一个用于“mywebsite”特定内容）。
2. 不必传递 Logger 对象引用。

该系统在运行时可配置。这种配置机制允许人们在不修改应用程序本身的情况下，调整记录级别和类型。

---

### 动机

如果在标准库中存在单一的日志记录机制，那么会有以下好处：

1. 日志记录更有可能“很好地”完成。
2. 多个库将能够集成到更大的应用程序中，这些应用程序可以合理地连贯地进行日志记录。

---

### 影响

该提案在研究了一下日志库后被提出：

- java.util.logging in JDK 1.4
- log4j
- the Syslog package from the Protomatter project
- MAL’s mx.Log package

---

### 简单的示例

这显示了一个非常简单的示例，说明了如何使用日志包来生成STDERR上的简单记录输出。

```python
--------- mymodule.py -------------------------------
import logging
log = logging.getLogger("MyModule")

def doIt():
        log.debug("Doin' stuff...")
        #do stuff...
        raise TypeError, "Bogus type error for testing"
-----------------------------------------------------
```

```python
--------- myapp.py ----------------------------------
import mymodule, logging

logging.basicConfig()

log = logging.getLogger("MyApp")

log.info("Starting my app")
try:
        mymodule.doIt()
except Exception, e:
        log.exception("There was a problem.")
log.info("Ending my app")
```

```
$ python myapp.py

INFO:MyApp: Starting my app
DEBUG:MyModule: Doin' stuff...
ERROR:MyApp: There was a problem.
Traceback (most recent call last):
        File "myapp.py", line 9, in ?
                mymodule.doIt()
        File "mymodule.py", line 7, in doIt
                raise TypeError, "Bogus type error for testing"
TypeError: Bogus type error for testing

INFO:MyApp: Ending my app
```

上面的示例显示默认输出格式。输出格式的所有方面都应配置，以便您可以这样格式化：

```
2002-04-19 07:56:58,174 MyModule   DEBUG - Doin' stuff...

or just

Doin' stuff...
```

---

### 控制流程

应用程序通过调用 Logger 对象来进行日志记录。Logger 被组织在了一个层次型命名空间（hierarchical namespace）中，在命名空间里，子 Logger 会从它们的父级节点上继承一些日志属性。

日志记录器名称符合“点分名称”命名空间，其中点（句点）表示子命名空间。因此，日志记录器对象的命名空间对应于单个树形数据结构。

- "" 表示命名空间的根节点
- "Zope" 是根节点的子节点
- "Zope.ZODB" 是 "Zope" 的子节点

这些 Logger 对象门会创建 **LogRecord** 对象，把他们传递给 **Handler** 对象进行输出。Loggers 和 Handlers 都可以使用日志级别和过滤器（可选）来确定它们是否对某条特定的 LogRecord 感兴趣。当有必要将 LogRecord 输出到外部时，Handler 可以在发送到 I/O 流之前，使用 Formatter（格式化器）进行本地化或者格式化消息。

每一个 Logger 都会跟踪一组输出 Handlers。默认情况下，所有 Loggers 将输出发送给它们祖先 Loggers 的所有 Handlers。但是，Loggers 也可以被配制成忽略高层树节点的 Handler。

这些 API 的结构化设计使得在禁用日志记录时，对 Logger API 的调用成本较低。如果给定的日志级别被禁用，那么 Logger 可以进行低成本的比较测试并返回。如果给定的日志级别已启用，Logger 仍会谨慎行事，在将 LogRecord 传递给 Handlers 之前尽量减少成本。特别是，本地化和格式化（这两者相对昂贵）会延迟到处理程序请求时才进行。

整个 Logger 层次结构也可以关联一个级别，该级别优先于各个 Logger 的级别。这是通过一个模块级函数来实现的：

```python
def disable(lvl):
    """
    Do not generate any LogRecords for requests with a severity less
    than 'lvl'.
    """
    ...
```

---

### 日志级别

日志级别，按照日志信息重要性从低到高排序，如下：

- DEBUG
- INFO
- WARN
- ERROR
- CRITICAL

术语 CRITICAL 是用来代替 FATAL 的，后者被 log4j 使用。这两个级别在概念上是相同的——即严重的或非常严重的错误。然而，FATAL意味着死亡，这在Python中意味着引发和未捕获的异常、跟踪和退出。由于日志模块并不强制要求 FATAL 级别的日志条目产生这样的结果，所以使用 CRITICAL 比 FATAL 更有意义。

这些日志级别只是简单的 integer 常数，为了去比较它们之间的重要程度。经验表明，过多的级别可能会导致困惑，因为它们导致对任何特定日志请求的级别的主观解释。

尽管强烈建议使用上述水平，但记录系统不应具有规定性。用户可以定义自己的级别以及任何级别的文本表示。但是，用户定义的级别必须遵守它们都是正整数的约束，并且按严重程度增加的顺序增加。

用户定义的记录级别通过两个模块级函数支持：

```python
def getLevelName(lvl):
        """Return the text for level 'lvl'."""
        ...

def addLevelName(lvl, lvlName):
        """
        Add the level 'lvl' with associated text 'levelName', or
        set the textual representation of existing level 'lvl' to be
        'lvlName'."""
        ...
```

---

### Loggers

每一个 Logger 对象都追踪它们感兴趣的级别的日志，并且丢弃那些比该级别低级的日志请求。

一个 Manager 类的对象维护了以名称作为层次型命名空间的 Logger 对象门。根据句点分割名称来表示不同的层次：Logger "foo" 是 Logger "foo.bar" 和 "foo.baz" 的父级节点。

Manager 类的实例其实是一个单例对象，并且不直接暴露给用户，用户使用各种模块级别的函数来和它交互。

一般的日志方法是：

```python
class Logger:
    def log(self, lvl, msg, *args, **kwargs):
        """Log 'str(msg) % args' at logging level 'lvl'."""
        ...
```

然而，每个日志级别都定义了更方便的函数：

```python
class Logger:
    def debug(self, msg, *args, **kwargs): ...
    def info(self, msg, *args, **kwargs): ...
    def warn(self, msg, *args, **kwargs): ...
    def error(self, msg, *args, **kwargs): ...
    def critical(self, msg, *args, **kwargs): ...
```

目前只识别一个关键字参数——"exc_info"。如果为真，调用者希望在日志输出中提供异常信息。只有当异常信息需要在任何日志级别提供时，才需要这种机制。在更常见的情况下，只有当错误发生时才需要将异常信息添加到日志中，即在ERROR 级别，那么就提供了另一个方便的方法：

```python
class Logger:
    def exception(self, msg, *args): ...
```

这应该只在异常处理程序的上下文中调用，而且是表示希望在日志中获得异常信息的首选方式。其他的简便方法只有在不寻常的情况下才可以和 exc_info 一起调用，例如，你可能想在 INFO 消息的上下文中提供异常信息。

上面显示的 "msg "参数通常是一个格式字符串；但是，它可以是 str(x) 返回格式字符串的任何对象 x。例如，这有利于使用一个对象为一个国际化/本地化的应用程序获取一个本地特定的消息，也许使用标准的gettext模块。一个简要的例子：

```python
class Message:
    """Represents a message"""
    def __init__(self, id):
        """Initialize with the message ID"""

    def __str__(self):
        """Return an appropriate localized message text"""

...

logger.info(Message("abc"), ...)
```

为一条日志消息收集和格式化数据可能会很昂贵，而且如果记录器无论如何都会丢弃该消息的话，那就是一种浪费。要想知道一个请求是否会被记录仪接受，可以使用isEnabledFor()方法。

```python
class Logger:
    def isEnabledFor(self, lvl):
        """
        Return true if requests at level 'lvl' will NOT be
        discarded.
        """
        ...
```

所以，与其像下面这样开销昂贵且可能会进行无用的 DOM 到 XML 格式转换：

```python
...
hamletStr = hamletDom.toxml()
log.info(hamletStr)
...
```

不如写成下面这样子：

```python
if log.isEnabledFor(logging.INFO):
    hamletStr = hamletDom.toxml()
    log.info(hamletStr)
```

当创建新的 Logger 时，它们会以表示“无级别”（no level）的级别进行初始化。可以使用setlevel（）方法明确设置级别：

```python
class Logger:
    def setLevel(self, lvl): ...
```

如果未设置 Logger 的级别，系统会咨询其所有祖先，遍历其层次结构，直到找到明确的设置级别。这被认为是该 Logger  的“有效级别”，可以通过 getEffectiveLevel() 方法来查询：

```python
def getEffectiveLevel(self): ...
```

Loggers 永远都不该被直接实例化，相反，应该使用模块级别的函数：

```python
def getLogger(name=None): ...
```

如果未指定名称，则返回根 Logger，如果存在该名称的 Logger，则将其返回。如果没有找到该名称的 Logger，则将新的Logger 初始化并返回。在这里，“名称”是“频道名称”的代名词。 用户可以在实例化新记录器时指定系统使用的自定义子类：

```python
def setLoggerClass(klass): ...
```

被传递的类应该是 Logger 的子类，其`__init__` 方法应该调用 `Logger.__init__`。

---

### Handlers

Handlers 负责将给定的 LogRecord 进行一些有用的处理，下面是一些核心的，将被实现的 Handlers：

- StreamHandler: 用于写入类似文件的对象的 Handler。
- FileHandler: 用于写入单个文件或一组轮转文件的 Handler。
- SocketHandler: 用于向远程 TCP 端口写入的 Handler。
- DatagramHandler: 写入 UDP 套接字的 Handler，用于低成本的日志记录。Jeff Bauer 已经有这样一个系统。
- MemoryHandler: 在内存中缓冲日志记录，直到缓冲区满了或出现特定条件。
- SMTPHandler: 通过 SMTP 协议发送邮件的 Handler。
- SysLogHandler: 用于通过 UDP 向 Unix syslog 写入的 Handler。
- NTEventLogHandler: 用于向Windows NT, 2000和XP的 event logs 写入的 Handler。
- HTTPHandler: 用于以 GET 或 POST 的方式向 Web 服务器写入信息的 Handler。

Handler 还可以使用 setLevel() 方法为他们设置级别：

```python
def setLevel(self, lvl): ...
```

可以设置 FileHandler 以创建一组轮转的日志文件集。在这种情况下，传递给构造函数的文件名被视为“基础”文件名。旋转的其他文件名是通过附加.1，.2等来创建基本文件名的其他文件名，最高为最大值，如要求汇总时指定的最大值。 setRollover 方法用于指定日志文件的最大大小和轮转中的最大备份文件数量。

```python
def setRollover(maxBytes, backupCount): ...
```

如果将 maxBytes 指定为 0，则永远不会发生轮转，并且日志文件无限期地增长。如果指定了非 0 大小，则当该大小将超过该大小时，会发生轮转。轮转方法可确保基本文件名称始终是最新的，.1是次新的，.2是之后次新的，以此类推。 在提供[6]的测试/示例脚本中还实现了许多其他处理程序 - 例如，XMLHandler和Soaphandler。

---

### LogRecords

一个 LogRecord 用于表示一个 logging 事件的容器。它不仅仅只是一个字典，尽管它确实定义了 getMessage 函数用于将一个消息与可选的运行参数合并。

---

### Formatters

一个 Formatter 用于将一个 LogRecord 转换成字符串类型来表示。一个 Handler 可能会在写入一个日志记录前调用 Formatter。以下核心的 Formatters 将会被实现：

- Formatter：提供类似于 printf 的格式化器，使用 % 操作符。
- BufferingFormatter：为多个消息提供格式化支持，并支持头部和尾部的格式化。

在 Handlers 对象上调用 setFormatter()，可以将 Handler 和 Formatter 连接起来：

```python
def setFormatter(self, form): ...
```

Formatters 使用 % 来格式化日志信息。格式化字符串应该包含 `%(name)s`，这些 LogRecord 属性字典可以用来获取特定信息的数据，以下属性将被提供：

| `%(name)s`            | Name of the logger (logging channel)                         |
| --------------------- | ------------------------------------------------------------ |
| `%(levelno)s`         | Numeric logging level for the message (DEBUG, INFO, WARN, ERROR, CRITICAL) |
| `%(levelname)s`       | Text logging level for the message (“DEBUG”, “INFO”, “WARN”, “ERROR”, “CRITICAL”) |
| `%(pathname)s`        | Full pathname of the source file where the logging call was issued (if available) |
| `%(filename)s`        | Filename portion of pathname                                 |
| `%(module)s`          | Module from which logging call was made                      |
| `%(lineno)d`          | Source line number where the logging call was issued (if available) |
| `%(created)f`         | Time when the LogRecord was created (`time.time()` return value) |
| `%(asctime)s`         | Textual time when the LogRecord was created                  |
| `%(msecs)d`           | Millisecond portion of the creation time                     |
| `%(relativeCreated)d` | Time in milliseconds when the LogRecord was created, relative to the time the logging module was loaded (typically at application startup time) |
| `%(thread)d`          | Thread ID (if available)                                     |
| `%(message)s`         | The result of record.getMessage(), computed just as the record is emitted |

如果 formatter 看到格式字符串包括"(asctime)s"，创建时间就会被格式化为 LogRecord 的 asctime属 性。为了允许灵活地格式化日期，formatter 被初始化为整个消息的格式字符串，以及一个单独的日期/时间格式字符串。日期/时间格式字符串应该是time.strftime格式。消息格式的默认值是"%(message)s"。默认的日期/时间格式是ISO8601。

formatter 使用一个类属性 "converter"，以表明如何将时间从秒转换为元组。默认情况下，"converter "的值是 "time.localtime"。如果需要，可以在单个 formatter 实例上设置一个不同的转换器（例如 "time.gmtime"），或者改变类属性以影响所有 formatter 实例。



