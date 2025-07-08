---
title: "Python简单脚本的日志记录实践"
date: 2025-07-08T16:15:00+08:00
summary: "Python 在简单场景下，日志的最佳实践"
---

## 目录

[TOC]

---

## 前言

很多一次性脚本或者经常运行的运维脚本，代码量往往很少，逻辑不复杂，并不需要配置过于细致复杂的日志系统。

但是，仅仅用 print 打印所有消息又过于简陋，所以本文提供一种在简单场景下，Python 脚本配置日志的实践。

默认情况下，logging 的根记录器（root logger）是有自己的默认配置的（level 是 warning 级别 + stderr 输出），并不适用于大部分脚本的场景，所以还是需要简单的日志配置。

---

## 控制台

最简单的单文件脚本，运行时间短，执行任务简单，代码量较少，直接在终端运行，不需要保存日志到文件中，仅仅在终端查看日志信息。

示例如下：

```python

import logging

logging.basicConfig(
	level=logging.DEBUG,
	format='%(asctime)s - %(levelname)s :: %(message)s'
)


def task(num):
	"""
	执行一个简单的累加任务并在最后除以一个随机数字
	"""
	import random
	logging.debug(f'task param <num> is {num}')
	
	result = 0
	for i in range(1, num+1):
		result += i
		logging.debug(f'task loop {i}, current result is {result}')

	# 随机制造一个除零异常，演示异常情况日志输出
	random_num = random.choice([0, 1, 2])
	logging.debug(f'task random_num is {random_num}')

	return result / random_num


def main():

	logging.info('main task start!')
	
	task_res = None
	try:
		task_res = task(4)
	except Exception as e:
		logging.exception(f'main task occur: {e}')
	
	if task_res:
		logging.info(f'main task got final result is {task_res}')
	else:
		logging.warning(f'task result is None.')


	logging.info('main task end...')


if __name__ == '__main__':
	main()

```

在简单场景下，使用 root logger 记录日志就够用了，basicConfig 就是用来配置 root logger 的。

通过指定 basicConfig 的 level 参数，可以控制输出不同级别的日志信息。

format 这里由于仅仅使用 root logger，所以我省略了 %(name)s 这个日志记录器名称字段，如果存在多个日志记录器，可以加上它。

except 块中，可以使用 logging.exception，效果等同于 logging.error('xxx', exc_info=True)，除了打印日志之外，还会打印错误堆栈信息。

---

## 日志文件

如果不需要把日志信息输出到控制台，而是全部存到本地日志文件中，则需要在 basicConfig 中指定几个参数。

basicConfig 配置如下:

```python
logging.basicConfig(
	level=logging.DEBUG,
	format='%(asctime)s - %(levelname)s :: %(message)s',
	filename='app.log',
	filemode='a',
)
```

filename 指定日志文件的名称，filemode 表示使用何种模式打开日志文件，默认模式为 'a'。

需要注意，filename 和 stream 参数不兼容，也就是说，指定了 filename，控制台的日志信息就输出不了了。

另外，basicConfig 不会自动创建 filename 父级目录，需要自己确保目标文件的父级目录存在并且权限上可以读写。

可以通过 Path 或者 os 模块，来确保父级目录是存在的:

```python
from pathlib import Path

log_dir = Path('./logs')
log_dir.mkdir(parents=True, exist_ok=True)


logging.basicConfig(
	level=logging.DEBUG,
	format='%(asctime)s - %(levelname)s :: %(message)s',
	filename=log_dir / 'app.log',
	filemode='a',
)
```

目录不存在会爆出`FileNotFoundError`，而权限不允许则会爆出`PermissionError`，所以要确保日志文件可读写。

---

## 控制台+日志文件

上面的两个例子分别给出了日志单独写入到控制台和单独写入到日志文件的案例，如果两者都要输出，那么需要额外的配置。

basicConfig 的 stream 和 filename 不能同时指定，否则会爆出:

```
ValueError: 'stream' and 'filename' should not be specified together
```

所以，只能通过另一个参数——handlers来手动指定多个目标handler。

```python
stream_handler = logging.StreamHandler()
file_handler = logging.FileHandler(filename='app.log', mode='a', encoding='utf-8')

logging.basicConfig(
	level=logging.DEBUG,
	format='%(asctime)s - %(levelname)s :: %(message)s',
	handlers=(stream_handler, file_handler),
)
```

这里手动创建两个 handler 对象，然后添加到 basicConfig 的 handlers 参数中。

---

## basicConfig vs 代码配置

上面的几个例子全部使用了 basicConfig 进行配置，当然也可以直接获取 root logger，然后对 handler，formatter，logger 逐个进行配置。

把上一小结的 basicConfig 转换成代码配置:

```python
# 获取 root logger 对象引用
root_logger = logging.getLogger()

# 创建 stream 和 file handler 的对象
stream_handler = logging.StreamHandler()
file_handler = logging.FileHandler(filename='app.log', mode='a', encoding='utf-8')

# 创建 formatter 对象
formatter = logging.Formatter('%(asctime)s - %(levelname)s :: %(message)s')

# 设置两个 handler 的 formatter
stream_handler.setFormatter(formatter)
file_handler.setFormatter(formatter)

# 把两个 handler 添加到 root logger
root_logger.addHandler(stream_handler)
root_logger.addHandler(file_handler)

# 设置日志级别
root_logger.setLevel(logging.INFO)

```

虽然最后效果是一样的，但是代码看起来远远超过了 basicConfig，basicConfig 封装了创建对象，以及构建对象关系的代码，看起来更加简洁。

在实际编程过程中，更推荐 basicConfig 配置。

