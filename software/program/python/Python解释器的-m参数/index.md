---
title: "Python解释器的-m参数"
date: 2025-09-13T09:11:00+08:00
summary: "-m参数到底做了什么事情？"
---

## 目录

[TOC]

---

## 前言

Python 解释器有一个参数 -m 经常使用，但是一直处于“不知其所以然”的状态，在使用虚拟环境和 pip 安装依赖包时，曾敲下过无数次包含 -m 参数的命令：

```sh
python -m venv .venv
python -m pip install requests
```

所以本文梳理下 -m 参数到底做了什么事情，以及在什么场景下使用该参数。

---

## 核心概念

python 的 -m 参数，后面是模块的名称，该参数告诉解释器，在 sys.path 中搜索用户指定的模块，把该模块当作脚本来执行。

---

## 与普通脚本的区别

假设现在一个脚本，路径是：/home/koril/Desktop/project/script/main.py

```python
import sys from pprint import pprint
pprint(sys.path)
```

比较执行该脚本的两种方式，第一种，以普通脚本文件的方式执行：

```sh
python /home/koril/Desktop/project/script/main.py
```

执行结果：

```
['/home/koril/Desktop/project/script',
 '/usr/lib/python312.zip',
 '/usr/lib/python3.12',
 '/usr/lib/python3.12/lib-dynload',
 '/usr/local/lib/python3.12/dist-packages',
 '/usr/lib/python3/dist-packages']
```

第二种，以模块的方式执行：

```sh
cd /home/koril/Desktop/project
python -m script.main
```

执行结果：

```
['/home/koril/Desktop/project',
 '/usr/lib/python312.zip',
 '/usr/lib/python3.12',
 '/usr/lib/python3.12/lib-dynload',
 '/usr/local/lib/python3.12/dist-packages',
 '/usr/lib/python3/dist-packages']
```

可以看出，以脚本文件的方式运行，添加到 sys.path 的路径是 /home/koril/Desktop/project/script，也就是脚本文件的父级目录，而以模块方式执行的结果显示，添加到 sys.path 的路径是 /home/koril/Desktop/project，表示模块的父级目录（也就是当前shell的所在目录）。

所以，除了添加到 sys.path 开头的路径不同之外，二者是很相似的。

---

## 包

如果传入的参数不是模块，而是一个包名，那么 python 会尝试运行包下的 __main__.py（如果存在的话）。

在 /home/koril/Desktop/project/script 下，编写 __main__.py:

```python
print('this is __main__')
```

执行：

```sh
cd /home/koril/Desktop/project
python3 -m script
```

---

## 使用场景

### 运行内置模块

这是最最常见的做法，Python 有很多实用的内置模块，在大部分变成场景下，这些模块都是以 import 的形式导入到我们的脚本文件中使用。

但是 -m 参数给了这个模块另外一种执行方式，直接在命令行下使用该模块。

考虑一个简单的场景，我想通过 Python 的 uuid 模块得到一个随机的 UUID 值。无论是 REPL 还是编写脚本，都需要导入 uuid 模块来使用：

```python
import uuid
print(uuid.uuid4())
```

通过 uuid 的源码可以看到，uuid 提供了 if  __name__ == '__main__' 块，意味着它可以作为脚本执行，并且代码里通过 argparse 解析用户传入的参数：

```sh
koril@ThinkBook:~/Desktop/project$ python3 -m uuid -h
usage: uuid.py [-h] [-u {uuid1,uuid3,uuid4,uuid5}] [-n NAMESPACE] [-N NAME]

Generates a uuid using the selected uuid function.

options:
  -h, --help            show this help message and exit
  -u {uuid1,uuid3,uuid4,uuid5}, --uuid {uuid1,uuid3,uuid4,uuid5}
                        The function to use to generate the uuid. By default uuid4 function is used.
  -n NAMESPACE, --namespace NAMESPACE
                        The namespace is a UUID, or '@ns' where 'ns' is a well-known predefined UUID addressed by namespace name. Such
                        as @dns, @url, @oid, and @x500. Only required for uuid3/uuid5 functions.
  -N NAME, --name NAME  The name used as part of generating the uuid. Only required for uuid3/uuid5 functions.

koril@ThinkBook:~/Desktop/project$ python3 -m uuid -u uuid4
772b97d4-89a5-4802-9fab-6a2b3911f335

```

由于 -u 有默认值（uuid4），所以直接可以在命令行中非常简单的获取一个随机的 uuid4 值：

```sh
python -m uuid
```

除了 uuid 之外，http.server 也很实用，它可以快速在当前目录运行一个 http 服务器：

```sh
python -m http.server
```

默认绑定 0.0.0.0 的 8000 端口，可以通过参数修改。

timeit 模块可以用来测量命令的执行时间，除了这几个外，还有很多有用的命令，不再一一赘述。

### pip

在一台机器上可能存在多个 Python 版本，3.7、3.8、3.11 等等，每个版本的 Python 可能都安装了一个 pip，仅仅在命令行中执行：

```sh
pip install xxx
```

无法知道是哪一个版本的 pip，所以推荐写法是：

```
python3.7 -m pip install xxx
python3.11 -m pip install xxx
```

这样可以明确 pip 属于哪一个版本。

### 包

-m 的另外一个重要用途体现在日常项目开发的包内模块测试问题。

考虑以下项目的目录结构：

```
my_project/
│
├── main_package/
│   ├── __init__.py
│   └── main_module.py
│
└── scripts/
    ├── __init__.py
    └── runner.py

```

runner.py 中导入了 main_module.py 中的内容：

```python
from main_package import main_module
```

假设 shell 处于 my_project 目录下，直接执行 python scripts/runner.py，会爆出 main_package 找不到的错误，因为 python 把 runner.py 的父级目录 my_project/scripts 添加到了 sys.path 中，而 main_package 显然不在 scripts 目录下，所以无法找到。

这时候就可以使用 -m 参数：python -m scripts.runner。

-m 参数模拟了真实的导入环境，之前甚至看到同事想要测试 runner.py，为了以脚本方式执行，绕开 ModuleNotFoundError 的问题，直接把整个 main_package 复制到了 scripts 目录下，执行成功，最后他来一句：别管歪门邪道，你就说，它成没成功吧。。。

---

## 小结

本文主要是为了针对 python -m 的知识进行查漏补缺，作为日常最常用的参数，知其然更要知其所以然。

---

## 参考

1. https://docs.python.org/3.13/using/cmdline.html#cmdoption-m
2. https://stackoverflow.com/questions/50821312/what-is-the-effect-of-using-python-m-pip-instead-of-just-pip
3. https://blog.dailydoseofds.com/p/python-m-the-coolest-python-flag
4. https://snarky.ca/why-you-should-use-python-m-pip/
