---
title: "Myers差分算法"
date: 2025-06-01T10:44:00+08:00
summary: "Myers 算法的原理，可视化，代码实现与应用"
---

## 目录

[TOC]

---

## 前言

日志记录，这是一项刚刚开始学习编程时，被我忽视的一个重要技能，因为初学者在开始学习时，往往是带有目的性或者任务性的编码，完成一个比较简单而具体的线性任务，所以日志记录，往往显得有些累赘，在编写一些即时使用，用后即弃的小脚本时，我们在脑海中清楚的知道代码的每一个过程，就算没有运行成功或者出现了一些异常，也都在掌控范围之内。

但是，当我开始编写了越来越多的大型软件，尤其是后端业务复杂性比较高的项目，以及服务器中不间断运行的各种定时任务脚本的时候，我开始意识到，日志框架的重要性绝不亚于其他任何框架，大部分的大型项目能够平稳运行和及时排错，日志的功劳绝对居于榜首。

夸张的来讲，好的日志记录可以有效降低后期维护程序员患上抑郁症的概率。

正如同电影中，塑造一个反派最简单的方法是给他一个悲惨的童年，而塑造一个愤世嫉俗，满眼血丝的程序员只要给他一个拥有极其糟糕的日志记录的项目就行。

在现在任何一个领域，信息的收集和整理都是个人或者企业进行分析和预测的重要前期准备，同样的，日志信息如果打得好，我们就能剩下大量时间和人力来排除故障，并且还能收集到非常重要的，线上的一手资料。

日志之所以没有在初学时期引起较大关注，究其原因还是苦吃少了，玩具代码确实不值得记录什么，但是工作中，只要吃过几次亏，被同事的悄无声息的包裹异常坑过几次，就会意识到防御式编程和日志的重要性。

综上所述，日志的编写、收集和分析都很有意义，我认为无论哪一门语言领域的开发者，日志技巧掌握程度的精进，对帮助自己能力迈向下一个阶段或者构建一个庞大，稳定，复杂的工程项目有着重要的实践意义。

本文，将介绍 Python 中日志记录的基础。

---

## 官方的文档

logging 模块的作者是 Vinay Sajip，Python 官方的 logging 文档作者也是他，目前找到的可以参考的主要主要资料如下：

- [PEP 282 – A Logging System | peps.python.org](https://peps.python.org/pep-0282/)
- [日志指南 — Python 3.13.1 文档](https://docs.python.org/zh-cn/3/howto/logging.html)
- [日志专题手册 — Python 3.13.1 文档](https://docs.python.org/zh-cn/3/howto/logging-cookbook.html#logging-cookbook)
- [logging --- Python 的日志记录工具 — Python 3.13.1 文档](https://docs.python.org/zh-cn/3/library/logging.html#module-logging)

有关作者的访谈和博客：

- [PyDev of the Week: Vinay Sajip - Mouse Vs Python](https://www.blog.pythonlibrary.org/2022/10/31/pydev-of-the-week-vinay-sajip/)
- [Red Dove Consultants Limited](https://www.red-dove.com/)

---

## 为何记录

log 源自大航海时代的测速法，人们把一个带着绳结的木头（log）投到海里，来测量航行速度，然后把航海的日志信息记录到 logbook，后来软件里的日志也沿用了这个叫法。

日志，是为了记录应用程序和进程在运行过程中的各个时间节点的状态和信息，有助于后期 BUG 的排查以及性能的优化。

当我们面对一个轰轰作响，但是没有任何消息反馈的庞然大物，它像一个巨大的黑色铁皮盒子一样，对我们隐藏一切过程，它断断续续的运行了一段时间后，悄然无息的倒下了，除了留下一个烂摊子，还有老板的一句：你排查下原因。此外，你没有任何线索。

排查故障就像侦探破案，首先，你得有线索。没有日志记录，你无从下手。

---

## 谨慎记录

不是说每个地方都记录就是完美的，每一个业务代码紧跟着一个日志代码显然是脱裤子放屁，而且过量的、无用的嘈杂信息，并不比不记录任何信息要好到哪里去。

我们更需要的精确的，优质的日志记录。

---

## Print 走天下？

无论是任何一门语言，一开始我们没有接触到日志框架的时候，所有的信息都是靠 Print 输出到控制台上的，它确实承受了太多，包括它不该承担的那些责任。

为什么在正式项目中，不用 print 来完成所有记录，有以下几个原因：

### 灵活性

当我们在前台运行脚本或者项目时，print 仅仅是把信息打印到控制台上，我们大多数场景希望能够记录日志到本地文件中，那么就只能手写文件打开，写入日志的代码，过于麻烦。

就算 nohup 能够把标准输出和标准错误重定向输出到一个文件中（事实上，我在经历的几家公司里都见过这种用法），但是问题还是很大，怎么控制一个文件大小和轮转（超过某个大小或者固定时间间隔，自动地输出到新的日志文件），又怎么控制多种输出目标（除了文件外，还有邮件，钉钉，LogStash）呢？

### 格式化

print 打印一句信息固然简单，但是格式（包括时间，日志级别，进程和线程的 ID，模块名等等）呢？线上出现了问题，打印了一个信息，你迫切的想知道打印这个信息时的具体时间，以及是哪个模块打印的（尤其是一些项目里，几百个一模一样的 print('error')，绝对让你抓狂）。

print 没法帮你全局格式化，手动每一个 print 都敲格式化相关的代码很累，格式化能帮你定位到了具体的时间（什么时候发生的）和具体的空间（哪一个线程的哪一个模块记录的）。

### 日志级别

print 在级别上属于一刀切，如果只是用 print 来调试还好，正式环境上线了也许我们不需要这么多日志信息，尤其是那些调试时才使用的过于细琐的信息，你怎么关闭呢？一行行用注释符号来注释掉那些不用的 print？

日志框架提供了多种级别的日志信息，相当于对这个信息打上了一个名叫“级别”的标记，你可以在不同场景下开关或者过滤掉这些日志。

对一些你比较关注的级别（比如 error 级别）的日志，单独处理，记录到一个特殊文件中，或者发送邮件。而那些 debug 级别的日志，在线上可以关闭，仅仅打开 info 级别以上的日志，这是日志框架很常见的操作。

### 可配置性

print 无法提供全局的，或者局部性的配置，也无法读取配置文件来配置打印的效果。

但在日志框架中，可配置性是基础，无论是通过代码来配置，还是配置文件来配置，都提供了非常灵活的选择。

### 责任分明

尽管似乎我在细数 print 的种种不是，但实际上这是由于它承担了太多它不该承担的任务，print 只是给用户提供简单的标准输出流的信息打印，而专业的日志记录应该交给专业的日志框架来完成。

---

## logging 模块的初级认识和使用

Python 中提供了一个日志模块——logging，它能满足大多数的日志记录需求。

使用它很简单：

```python
import logging
logging.warning('warning...')
```

打印结果如下：

```
WARNING:root:warning...
```

这是默认格式，除了我们提供的信息之外，还有两个用英文冒号隔开的信息，第一个是日志级别，第二个是默认日志记录器（logger）的名字，也就是 root。

### 什么是日志级别

日志级别是日志记录器（稍后会讲解什么是日志记录器）用来给日志打的标记，来表示这一条日志的级别或者说严重程度，Python 的 logging 模块提供了以下的五种级别：

1. DEBUG：非常细节的信息，仅当诊断问题，排除故障时打开。
2. INFO：用来确认程序正在按照预期进行。
3. WARNING：程序正在按照预期进行，但是即将面临一些问题，比如：磁盘快不足了，网络有些波动。
4. ERROR：程序的某个功能出现了问题，无法正常使用，需要尽快排查，但程序还能继续运行。
5. CRITICAL：程序发生了严重错误，已经无法继续运行。

根据不同场景和需要，选择不同级别的记录，这是迈向好的日志记录的第一步。

但当你满怀期待的打算测试下每个级别的输出是，结果可能会偏离预期，执行下面的代码：

```python
def main():
    logging.debug('debug...')
    logging.info('info...')
    logging.warning('warning...')
    logging.error('warning...')
    logging.critical('warning...')
```

打印结果：

```
WARNING:root:warning...
ERROR:root:warning...
CRITICAL:root:warning...
```

可以发现 warning、error、critical 有打印信息，但是 debug 和 info 没有输出。

这是由于到目前为止，我们所用的日志记录器，是一个名为 root 的默认日志记录器，没有做任何配置和改动，无论是打印格式，输出日志的目标位置，还是日志级别都是默认的。

这里没有输出 debug 和 info 的信息，是因为默认 logger 的日志级别是 warning，按照级别从低到高排序：debug < info < warning < error < critical。当日志级别是 warning 的时候，日志只会输出 warning 以及 warning 以上的级别的信息，并且，默认的输出流目标是标准错误流，所以在终端中执行，会显示红色。

这就给代码的开发提供了很大的自由度，因为当你关心某一个级别的时候，大概率你也会关心比该级别更高的信息，如果你仅仅想要输出单个级别的信息，可以参考后面讲到的过滤器（Filter）配置。

再次注意，我们现在并没有创建一个自己的日志记录器，一切都是直接调用的 logging 模块的函数，也就是调用 root 记录器，root 记录器会在调用日志方法前，调用 basicConfig 函数（没有遵循 pep8 命名规则，是因为这个模块从 Java 的 log4j 参考过来的），我们可以通过 basicConfig 对默认的配置做出一些修改：

```python
# 设置 root 日志记录器的日志级别
logging.basicConfig(level=logging.DEBUG)

def main():
    logging.debug('debug...')
    logging.info('info...')
    logging.warning('warning...')
    logging.error('warning...')
    logging.critical('warning...')
```

### 格式化日志

格式化日志是日志框架中的一个重点，这也是它经过配置后远胜于 print 的一个地方。

root 记录器的默认格式是输出三个内容：日志级别，日志记录器名称，日志信息。

我们可以通过 basicConfig 的 format 参数来定义日志的格式：

```python
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
```

输出如下：

```
2024-09-16 10:16:22,508 - root - DEBUG - debug...
2024-09-16 10:16:22,508 - root - INFO - info...
2024-09-16 10:16:22,508 - root - WARNING - warning...
2024-09-16 10:16:22,508 - root - ERROR - warning...
2024-09-16 10:16:22,508 - root - CRITICAL - warning...
```

加了个打印了记录器输出的日志的时间，并且通过间隔符号和空格，使得整体看起来没那么拥挤了。

asctime 和 name 这些参数都是预先定义好的，完整的列表参考以下文档：

[logging — Logging facility for Python — Python 3.12.6 documentation](https://docs.python.org/3/library/logging.html#logrecord-attributes)

format 支持 python 提供的几种 String 格式化标准：%(name)s，${}, {}，仅需在 style 参数指定格式符号即可：

```python
# 以下几种效果是一样的，挑选其一即可，style 默认值是 %
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', style='%')
logging.basicConfig(format='${asctime} - ${name} - ${levelname} - ${message}', style='$')
logging.basicConfig(format='{asctime} - {name} - {levelname} - {message}', style='{')
```

如果想要自定义 asctime 的时间格式，还有个 datefmt 参数：

```python
# 去掉毫秒
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
)
```

datefmt参数的格式与 [`time.strftime()`](https://docs.python.org/zh-cn/3/library/time.html#time.strftime) 支持的格式相同。

### basicConfig 的位置

这里有个坑，如果你要自己调用 basicConfig，那么一定要在使用 debug，info 等等这些记录方法前调用，否则无效，

可以看下面的代码片段，虽然调用了 basicConfig，但是并没有起到作用：

```python
import logging

logging.debug('debug message')

logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
)

def main():
    logging.debug('debug...')
    logging.info('info...')
    logging.warning('warning...')
    logging.error('warning...')
    logging.critical('warning...')


if __name__ == '__main__':
    main()

```

### 持久化到文件

仅仅在控制台输出，仅适用于一些测试性质的玩具代码，正式项目基本上都需要持久化日志到文件，basicConfig 也可以通过一些配置，来达到这样的效果。

```python
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S',
    filename='app.log',
    filemode='w'
)
```

和 open() 类似，指定文件名字和模式，模式是 w 表示覆盖原有内容，模式是 a 表示追加内容。

但这里有个很怪的地方，我以为 basicConfig 可以指定 encoding，但是 basicConfig 并不接收这个参数，basicConfig 默认调用的 FileHandler 的 encoding 参数默认值是 None，所以如果要指定 encoding，只能通过后面介绍的 FileHandler 来处理，这很奇怪，参考以下帖子：

[python - Add encoding parameter to logging.basicConfig - Stack Overflow](https://stackoverflow.com/questions/10706547/add-encoding-parameter-to-logging-basicconfig)

### 记录异常

异常是常有的事，考虑以下代码：

```python
import logging

logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
)


def main():
    person = {
        'name': 'koril'
    }
    try:
        age = person['age']
        print(f'person age is {age}')
    except Exception as e:
        logging.error('something wrong')


if __name__ == '__main__':
    main()
```

显然，person 中没有 age 这个 key，所以会走到 except 块中，但问题是我们需要记录更详细的堆栈信息，一种不恰当的方法是打印 exception，这通常有点用，但不太多，我们更需要记录下的是异常的堆栈信息，第一种方法是使用普通的日志方法，并且指定参数 exc_info 为 True：

```python
try:
    # some code
except Exception:
    # 不一定是 error，这里也可以写 debug，info 之类的，它们都有 exc_info 可以指定
    logging.error(f'something wrong', exc_info=True)
```

另一种方法是使用 exception，exception 默认的级别就是 error，并且会记录异常的详细信息，需要注意的一点是，仅应该在异常处理中使用该方法。

上面的代码可以写成：

```python
try:
    # some code
except Exception:
    logging.exception(f'something wrong')
```

如果不在 except 块中使用 exception，则会多输出一个 NoneType: None，看起来很难受，所以最好不要这么干：

```python
def main():
    logging.exception(f'something wrong')

if __name__ == '__main__':
    main()
```

输出：

```
2024-09-16 11:32:16 - root - ERROR - something wrong
NoneType: None
```

---

## 初级认识的总结

了解完以上的内容，日志模块可以完成 90% 的测试性代码和玩具项目所需要的记录。

对于不同的需要，精确的选择日志级别，开发测试时，多打 debug，正常运行项目时，若需要观察一些关键点的状态和运行进度，使用 info，如果检测到可能出现的问题，但还不那么严重，使用 warning，如果某个流程出现问题，尽管程序还能继续运行，但已经需要人为介入的，则是 error 级别，出现了 critical 级别的日志，表示程序出现重大问题，已经不能继续正常运行了，意味着你要加班了。

在服务器后台上跑的小脚本，可能也就运行十天半个月的，可以把日志持久化到文件，在 basicConfig 中指定文件名，记得要选择模式 a，这样才能追加，而不是每次重启就覆盖掉原来的内容。

异常处理可以选择 exception，默认 error 级别，并且记录异常堆栈信息。

---

## 进阶

如果要在工程实践中，更好的使用日志，仅仅靠上面的内容是远远不够的，我们还要一些更加灵活和精细的技巧。

### 组件

日志库采用模块化方法，并提供几类组件：记录器、处理器、过滤器和格式器。

- 记录器（logger）：暴露了应用程序代码直接使用的接口。
- 处理器（handler）：将日志记录（由记录器创建）发送到适当的目标。
- 过滤器（filter）：提供了更细粒度的功能，用于确定要输出的日志记录。
- 格式器（formatter）：指定最终输出中日志记录的样式。

日志信息就是 LogRecord 的实例，它在以上提到的几个组件中按照某种顺序和规则传递。

### 记录器（logger）

到目前为止，我们所使用的记录器都是一个名叫 root 的根记录器，也就是默认的记录器，我们可以获取自定义的记录器：

```python
import logging

logger = logging.getLogger(__name__)
```

getLogger 接受一个参数（不传参的话，就是获取 root 记录器），表明记录器的名称，这个名称有很大作用，建议使用 `__name__` 作为传入参数，这样的话，有个好处，记录器和记录器之间的继承关系是依赖于这个名称的，例如：名为 scan 的记录器是名为 scan.html 和名为 scan.pdf 的记录器的父级，所以如果用 `__name__` 来做参数，可以很好的利用 Python 的包和模块的命名空间。

记录器有两类作用，一个是配置，一个是创建日志记录。

配置有以下几种：

1. 配置级别，setLevel，也就是之前提到的日志级别，如果不配置的话，则会向父级寻找，然后继承父级的日志级别。
2. 配置处理器，addHandler。
3. 配置过滤器，addFilter。

创建日志记录有以下几种：

1. debug，info，warning，error，critical 方法
2. 在异常处理块中使用的 exception 方法
3. log 方法，上面几种日志级别作为显示参数传入该方法，虽然冗长，但是这个方法在自定义日志级别时可以使用。

### 处理器（handler）

Handler 对象负责将适当的日志消息（基于日志消息的严重性过滤）分派给处理器的指定目标。

Logger 对象可以使用 addHandler 方法向自己添加零个或多个处理器对象。为什么需要多个处理器对象？因为一个日志消息可能会发给多个目标。

应用程序可能希望将所有日志消息持久化到日志文件，然后错误或更高的所有日志消息发送到标准输出流，最后将所有关键消息发送至一个邮件地址。这种方案需要三个单独的处理器，其中每个处理器负责将特定严重性的消息发送到特定位置。这一个特性，使得我们对单个消息的多种处理有了很大的自由度。

对每种不同的任务，日志框架都提供了对应的处理器类，总共有十多种，可以参考以下文档中的“有用的处理器”这一小节：

[日志指南 — Python 3.12.6 文档](https://docs.python.org/zh-cn/3/howto/logging.html#useful-handlers)

本文会提及以下几种：

* StreamHandler 实例发送消息到流（类似文件对象）
* FileHandler 实例将消息发送到硬盘文件
* BaseRotatingHandler 是轮换日志文件的处理器的基类。它并不应该直接实例化。而应该使用 RotatingFileHandler 或 TimedRotatingFileHandler 代替它。
* RotatingFileHandler 实例将消息发送到硬盘文件，支持最大日志文件大小和日志文件轮换
* TimedRotatingFileHandler 实例将消息发送到硬盘文件，以特定的时间间隔轮换日志文件。
* HTTPHandler 实例使用 GET 或 POST 方法将消息发送到 HTTP 服务器。

如果我们仅仅希望打印日志到控制台，我们可以选用 StreamHandler：

```python
import logging

logger = logging.getLogger(__name__)
logger.setLevel(logging.DEBUG)

stream_handler = logging.StreamHandler()
stream_handler.setLevel(logging.DEBUG)
stream_handler.setFormatter(logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s'))

logger.addHandler(stream_handler)


def main():
    logger.debug('debug...')
    logger.info('info...')
    logger.warning('warning...')
    logger.error('error...')
    logger.critical('critical...')


if __name__ == '__main__':
    main()

```

如果我们不仅希望写入到控制台，还希望写入到文件，则给 logger 加一个 FileHandler 的实例即可：

```python
import logging

logger = logging.getLogger(__name__)
logger.setLevel(logging.DEBUG)

# 输出日志到控制台
stream_handler = logging.StreamHandler()
stream_handler.setLevel(logging.DEBUG)
stream_handler.setFormatter(logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s'))
# 输出日志到文件
file_handler = logging.FileHandler('app.log', mode='a', encoding='utf-8')
file_handler.setLevel(logging.WARNING)
file_handler.setFormatter(logging.Formatter('%(asctime)s - %(message)'))

logger.addHandler(stream_handler)
logger.addHandler(file_handler)


def main():
    logger.debug('debug...')
    logger.info('info...')
    logger.warning('warning...')
    logger.error('error...')
    logger.critical('critical...')


if __name__ == '__main__':
    main()

```

我们这里加一个 FileHandler，跟 basicConfig 不同，它可以指定 encoding，不同的 handler 之间的级别和格式不必相同，向上面的例子里，stream_handler 的级别是 debug，意味着它将打印所有日志信息到控制台，而 file_handler 则是 warning 级别，文件里仅会记录 warning 及以上的日志信息，并且两个 handler 的格式也不同。

以上，我们添加了两个处理器，但是如果没有添加处理器会是什么行为呢？

以下代码，仅仅获取了一个 logger，但是没有配置任何处理器：

```python
import logging

logger = logging.getLogger(__name__)


def main():
    logger.debug('debug...')
    logger.info('info...')
    logger.warning('warning...')
    logger.error('error...')
    logger.critical('critical...')


if __name__ == '__main__':
    main()

```

输出：

```
warning...
error...
critical...
```

可以发现，没有任何格式，仅有信息，而且 debug 和 info 级别的信息没有输出。这里日志框架提供了一个兜底的处理器，也叫做“最后的处理器”，它存储在 logging.lastResort 中。

这个内部处理器与任何日志记录器都没有关联，它的作用类似于 StreamHandler，它将消息写入到 sys.stderr，并且没有对消息进行任何格式化 —— 只打印简单的事件描述消息。 

该处理器的级别被设为 WARNING，因此将输出严重性在此级别以上的所有事件。

### 有两个 setLevel？

上面的代码我们可以看到 logger 设置了一次 level，而两个 handler 又各自设置了一个 level，在记录器和处理器上设置的 level 的区别在于，记录器的 level 像是个总闸，低于记录器设置级别的日志是不会流到处理器那里的，换句话说，处理器只会处理那些比记录器等级要高的日志。

处理器也可以设置 level，那么低于处理器 level 的日志就不会流向目标。

如果 logger 不设置 level，那么 logger 会一直向父级寻找，知道找到设置了 level 的父级，然后继承它的 level，如果每一个父级都没设置，那么最后找到根记录器，根记录器是一定有 level 的，那就是 warning。

如果 handler 不设置 level，那么 handler 会从 logger 那继承 level。

---

### 过滤器

目前为止，我们所有的过滤都是基于日志级别的，当我们设置了某个 level 后，等于或者大于该 level 的日志信息会被记录，这种方式适用于绝大部分情况，但是还有些需求，我们需要借助过滤器（Filter）来实现更加细粒度的控制。

考虑这种需求：我们仅仅需要打印所有的 DEBUG 级别的信息，DEBUG 以上的都忽略。

```python
import logging

logger = logging.getLogger(__name__)
logger.setLevel(logging.DEBUG)

console_handler = logging.StreamHandler()
console_handler.setLevel(logging.DEBUG)
console_handler.setFormatter(logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s'))

# 重点在这里，可以通过加一个过滤器来过滤掉不是 debug 级别的日志
console_handler.addFilter(lambda record: record.levelno == logging.DEBUG)

logger.addHandler(console_handler)


def main():
    logger.debug('debug message')
    logger.info('info message')
    logger.warning('warning message')
    logger.error('error message')
    logger.critical('critical message')


if __name__ == '__main__':
    main()

```

addFilter 接收一个函数，函数的入参是 LogRecord 实例，返回值是 True or False，所以上面我用 lambda 表达式来编写了这个过滤器。

LogRecord 有很多参数可以用来判定是否进行过滤，参考以下文档：

[logging — Logging facility for Python — Python 3.12.6 documentation](https://docs.python.org/3/library/logging.html#logrecord-objects)

---

## 从文件或者字典中进行日志的配置

以上所有的配置都是基于 Python 代码完成的，虽然书写时很方便（过程式的代码配置），但是后期维护、阅读和修改就没那么简洁明了了，这时候我们就可以使用配置文件来代替代码配置。

### 日志配置文件

我们可以将上面的代码转换成一下 conf 文件，文件格式是 ini：

```ini
[loggers]
keys=root,app

[handlers]
keys=consoleHandler,consoleHandlerRoot

[formatters]
keys=simpleFormatter,rootFormatter

[logger_root]
level=DEBUG
handlers=consoleHandlerRoot

[logger_app]
level=DEBUG
handlers=consoleHandler
qualname=app
propagate=0

[handler_consoleHandler]
class=StreamHandler
level=DEBUG
formatter=simpleFormatter
args=(sys.stdout,)

[handler_consoleHandlerRoot]
class=StreamHandler
level=DEBUG
formatter=rootFormatter
args=(sys.stdout,)

[formatter_simpleFormatter]
format=%(asctime)s - %(name)s - %(levelname)s - %(message)s

[formatter_rootFormatter]
format=root ---- %(asctime)s - %(name)s - %(levelname)s - %(message)s
```

这里配置了 root 和 app 两个 logger，为了显示出 propagate 的作用，我特地将二者的 handler 区分开来，并且两个 handler 的 formatter 不一样，如果没有 propagate 参数为 0，那么默认 app 会传播给 root，导致信息打印两次。

读取配置文件的代码也很简单：

```python
logging.config.fileConfig('logger.conf')
logger = logging.getLogger('app')
```

### 字典配置

除了使用配置文件之外，官方文档还提供了另一种方法——字典配置，也是新的应用程序的推荐使用方法。

因为它使用 Python 的字典提供了一种中间层，你的配置文件可以是 ini，json，yaml 等种种格式，通过 Python 解析成字典，再把该字典导入 logging.config.dictConfig 方法。

字典配置必须要有的一个键是 version，值为 1，没有写的话，会爆以下错误：

```
ValueError: dictionary doesn't specify a version
```

这里的字典配置和上面的 ini 文件配置效果相同：

```python
my_config = {
    'version': 1,
    # 格式化器
    'formatters': {
        'simple_formatter': {
            'format': '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        }
    },
    # 处理器
    'handlers': {
        'console_handler': {
            'class': 'logging.StreamHandler',
            'formatter': 'simple_formatter',
            'level': logging.DEBUG,
            'stream': 'ext://sys.stdout'
        }
    },
    # 记录器
    'loggers': {
        'app': {
            'handlers': ['console_handler'],
            'level': logging.DEBUG,
            'propagate': False,
        }

    },
    # 根记录器
    'root': {
        'handlers': ['console_handler'],
        'level': logging.DEBUG
    }
}

logging.config.dictConfig(my_config)
logger = logging.getLogger('app')
```

这里的 root 记录器配置不是必要的，只是为了演示 propagate 的作用。

可以使用可读性更好的 yaml 文件作为配置文件，然后再转换成字典，传给 dictConfig：

```yaml
# 版本，不写的话会报错
version:
  1

# 格式化器
formatters:
  simple_formatter:
    format: '%(asctime)s - %(name)s - %(levelname)s - %(message)s'

# 处理器
handlers:
  console_handler:
    class: logging.StreamHandler
    level: DEBUG
    formatter: simple_formatter
    stream: ext://sys.stdout

# 记录器
loggers:
  app:
    level: DEBUG
    handlers: [console_handler]
    propagate: False

# 根记录器
root:
  level: DEBUG
  handlers: [console_handler]
```

对应的 Python 解析代码如下：

```python
# 需要安装 yaml 包：python -m pip install pyyaml
import yaml

my_config = yaml.load(open('logger.yaml', encoding='utf-8'), Loader=yaml.FullLoader)

logging.config.dictConfig(my_config)
logger = logging.getLogger('app')
```

yaml 文件的可读性比 Python 字典更好，少了逗号和花括号，唯一需要注意的就是 yaml 对缩进要求很严格。

除了 yaml 之外还可以选用其他格式的配置文件，最终只要能转换成 Python 字典就行。

---

## 异常还是隐藏？

如果你见过下面这种代码：

```python
from random import randint

num = None
try:
    # 模拟 num 通过一些可能会出现异常的复杂过程才能得到
    num = 1 / randint(0, 1)
except Exception:
    # 有时候，这里连一个打印甚至都不给，直接 pass
    print('没啥事儿，一切都好')

# 你有时候抓耳挠腮，排查了半天的 NPE 也找不到问题所在
print(f'执行成功, num: {num}')
```

你就能理解为何我写这一小节。







---

## 再议 basicConfig 的默认行为





---

## 日志的轮转







---

## 关注记录的大小

日志是一种辅助手段，但没有合理地使用时，它可能会成为应用的杀手，甚至影响到服务器和其他应用。

这里要讨论的是日志的大小，如果不关注日志的大小，它可能会像一颗定时炸弹一样，潜藏在服务器中，不知道什么时候就悄悄占满了服务器的磁盘空间（特别是在没有做服务器磁盘空间预警的时候）。

以前曾碰到两种情况：

1. nohup 写到一个文件里，文件随时间越来越大。
2. 单条日志过大， 比如记录了一个 post 请求，那个 body 大得很，1，2M 的单条记录只要记录个几千次，日志文件就很夸张了，还有一次记录了图片转 base64 编码的结果，为了排查一个编码的问题，一个图片二进制可能就 7，8M，转成 base64 更是变大了不少。

这里的教训有以下几点：

1. 上线前需要检查 logger 的情况，拒绝乱打 debug，info，为什么强调这俩哥们，因为大部分程序员很少会选择 warning 以上的，大家都无比钟爱 debug 和 info，所以很容易出现大量的没有用的 info 日志。
2. 上线后的日志级别选在 info 或者 warning 以上，不然大量的 debug 会淹没正常的信息。
3. 一定要使用日志滚动，有必要的时候使用日志压缩，很多网上的博客讲切割日志，采用的 bash 脚本 + crontab 定时任务的方法，我认为不是最优解，这种方法将应用和服务器耦合了，应用的日志问题，应该由应用自身来解决，而且日志的切割和滚动是大部分日志框架的基础行为，不要服务器脚本的介入。
4. 考虑将不同级别的日志分开存放，也可以考虑不同服务或者模块的日志分开存放。
5. 关注 error 和 critical 级别的日志，最好能第一时间通知（邮件，短信），很多问题一开始解决掉，就没后面那么多事儿了。

---

## 函数库中的日志







---

## 参考

1. [logging --- Python 的日志记录工具 — Python 3.12.6 文档](https://docs.python.org/zh-cn/3/library/logging.html#logging.basicConfig)
2. [日志指南 — Python 3.12.6 文档](https://docs.python.org/zh-cn/3/howto/logging.html)
3. [日志专题手册 — Python 3.12.6 文档](https://docs.python.org/zh-cn/3/howto/logging-cookbook.html)
4. [Logging in Python – Real Python](https://realpython.com/python-logging/)
5. [logging.config --- 日志记录配置 — Python 3.12.6 文档](https://docs.python.org/zh-cn/3/library/logging.config.html#)