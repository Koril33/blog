---
title: "命令行设计的艺术"
date: 2025-11-17T17:01:23
summary: "如果屏幕和键盘一直存在的话，CLI也许永远也不会被淘汰"
---

## 目录

[TOC]

---

## 前言

上世纪七、八十年代诞生的命令行工具在现在的用户看来似乎有些陈旧和繁冗：黑底白字，复杂的帮助手册，莫名其妙的报错，以及需要敲击命令所带来的较高门槛，CLI（Command Line Interface）远远没有 GUI（Graphical User Interface）来的那么直观和平易近人。在 GUI 中，大部分的操作都是通过点击、拖拽、框选等鼠标动作来完成，在设计合理的情况下，能大幅降低使用者的学习复杂度。

但这并不意味着命令行工具会被取代，相反它会永远存在，因为相比 GUI，它更适合脚本化，自动化，以及能够以最低的资源占用完成非常复杂和多样的操作。所以 GUI 和 CLI 并非相互取代，而是要取决于你设计的工具产品的定位，如果一个产品面向大众或者非计算机领域的工程师，那么 GUI 的直观操作无疑是最好的选择，而面向是运维人员，软件开发者或者计算机爱好者，那 CLI 也许更加合适。

CLI 和 GUI 都是一种交互方式，它们需要封装的是计算机指令的复杂性，当我们为了完成一次图片的处理，比如：图像的二值化，对于底层来说是操作每个像素的值，等于或高于阈值的变成 1，低于阈值的变成 0，但是对于用户而言，他们不可能为了处理图片而专门学习计算机的编程语言，所以需要对业务逻辑进行封装。

GUI 会选择展示图片，并且提供菜单选项和按钮，让用户就像是去饭店点餐一样直观简单。

CLI 则通过终端，用户需要键入指令，传入的图片路径，操作命令，以及输出的图片保存路径。

市面上关于 GUI 的设计课程种类繁多，并且在实际生产中有单独的设计岗位（UI/UX）来完成这份工作。相较之下，对于 CLI 的设计指导的讨论度就没那么高了，因为往往都是工具核心的设计者（而不是一个单独的岗位）来完成交互界面的设计，设计的好坏都取决于软件开发者的品味。

大多数命令行工具都诞生于 Unix/Linux 之下，好在他们的开发者品味都不错，所以大部分的工具都经久不衰，皮实耐用，经历了几十年的生产实践的考验。

本文会整理一下关于 CLI 的设计哲学和规范，提供一种在设计各类工具 CLI 时的参考指南，这里提到的大部分建议都围绕界面设计而非代码层面的设计，即如何设计一个好用、易用的符合规范的 CLI 工具。

---

## 选择

在开发一个 CLI 工具的时候，首先要确认你的需求是否适合用 CLI 实现。

以下的需求适合 GUI：

1. 面向的群体是学生，设计师，普通用户，职员等非技术人员。
2. 涉及大量的图片、表格、图表、视频的展示和交互，比如图片视频编辑器，表格处理，数据可视化，网络电商等。
3. 专业领域的复杂软件，需要结合业务，降低使用门槛，比如：金融、医疗、基础建设、物联网等领域相关的软件。

以下的需求适合 CLI：

1. 面向的群体是程序员，技术人员，运维人员，计算机爱好者等能够习惯使用终端的用户。
2. 工具需要频繁在远程服务器运行，因为大部分运维人员面对的只有 SSH 连接终端，没有桌面。
3. 简单的工具类软件，完成单一、可靠的功能，适合作为流水线的一环，适合自动化和批处理，比如 Unix/Linux 下的大部分基础命令。
4. 复杂的工具类软件，需要大量的配置和精确控制，比如：git、ffmpeg、docker、nmap 等等。

---

## 以人为本

过去的很多命令是面向程序的，所以设计时很少考虑到人机交互，导致参数输入、结果输出、使用方式比较反人类。

但是这些命令已经成为了整个计算机行业的基石，承载了太多历史性的包袱，所以在设计新的命令行工具的时候，避免一味的模仿，作为 Interface，就必须要以用户的体验为中心。

如果底层实现比较复杂和晦涩，要注意避免需要用户深入学习才能使用工具，而是提供一定层次的封装和简化，就像是现实世界的水、电、燃气等基础建设资源，普通的用户不需要去深入了解能源的产生、转化和运输，只需要知道开关阀门就能获取到能源就行了。

---

## 示例、帮助和文档

无论是 GUI 和 CLI，为了教导用户如何使用这个工具，都需要提供帮助文档（help page）或者操作手册（manual）。

GUI 在这方面远胜 CLI，因为它能提供丰富的图片视频等交互资源（比如 Adobe 的示例）。

而在过去没有互联网的时候，CLI 只有一个 man 命令，有了互联网之后，大部分新手用户也会通过搜索引擎查看 web 文档。

CLI 工具不仅仅存在于 Unix/Linux 了，跨平台的需求下，把帮助内容和工具教程放在 web 文档更加合适，应该把主要精力放在维护 web 而非古老的 man page。

现代化的 CLI 工具，应该能够让用户快速获取到清晰易懂的示例和帮助文档，首先就是提供一个程序说明，以及能够运行的基础命令示例，让用户能够快速上手，确保工具的正常安装。

### 程序级别帮助

假设一个简单的下载器命令叫`idownload`，那么在键入以下内容时，应该显示程序级别的帮助文档：

```shell
idownload

idownload -h

idownload --help
```

程序级别的帮助文档应该包含以下内容：

1. 工具程序的名称和版本号
2. 主要的功能概述
3. 最基础的命令示例
4. 命令格式，选项、参数的说明
5. 如果有 github 或者 web 文档，可以把地址放在最后

Linux 下的 jq 就是一个不错的示例：

```shell
~$ jq
jq - commandline JSON processor [version 1.6]

Usage:	jq [options] <jq filter> [file...]
	jq [options] --args <jq filter> [strings...]
	jq [options] --jsonargs <jq filter> [JSON_TEXTS...]

jq is a tool for processing JSON inputs, applying the given filter to
its JSON text inputs and producing the filter's results as JSON on
standard output.

The simplest filter is ., which copies jq's input to its output
unmodified (except for formatting, but note that IEEE754 is used
for number representation internally, with all that that implies).

For more advanced filters see the jq(1) manpage ("man jq")
and/or https://stedolan.github.io/jq

Example:

	$ echo '{"foo": 0}' | jq .
	{
		"foo": 0
	}

For a listing of options, use jq --help.
```
用户键入 jq 并回车，得到了一个长度合适的帮助文档（过于复杂的帮助文档可以放在 web 上，或者使用终端分页程序展示）。

输出的第一行是 CLI 的名称和版本号，紧接着就是 CLI 的使用格式，在简单介绍了该 CLI 能做些什么后，提供了一个基本的示例，并且给出了 github 的地址。

### 子命令的帮助

很多大型 CLI 工具，譬如：git、docker，都有子命令的概念，这种子命令组成的树状结构是为了更好的分隔不同的业务功能。

除了提供程序级别的帮助文档之外，也需要对每一个子命令提供详细的说明。

用户键入以下内容，应该提示子命令的帮助文档：

```shell
idownlaod config -h

idownload config --help
```

这里，config 是一个子命令，通过 -h 或者 --help，可以获取到该子命令的帮助文档。

docker 的例子如下：

```shell
koril@ali-djhx-debian:~$ docker ps --help
Usage:  docker ps [OPTIONS]

List containers

Aliases:
  docker container ls, docker container list, docker container ps, docker ps

Options:
  -a, --all             Show all containers (default shows just running)
  -f, --filter filter   Filter output based on conditions provided
      --format string   Format output using a custom template:
                        'table':            Print output in table format with
                        column headers (default)
                        'table TEMPLATE':   Print output in table format using
                        the given Go template
                        'json':             Print in JSON format
                        'TEMPLATE':         Print output using the given Go
                        template.
                        Refer to https://docs.docker.com/go/formatting/ for
                        more information about formatting output with templates
  -n, --last int        Show n last created containers (includes all states)
                        (default -1)
  -l, --latest          Show the latest created container (includes all states)
      --no-trunc        Don't truncate output
  -q, --quiet           Only display container IDs
  -s, --size            Display total file sizes
```

---

## 选项、参数和子命令

对于 CLI 的交互而言，最重要的就是输入，选项、参数以及子命令是不同的输入类型，它们之间有着明确的语义上的区别。

### 参数

参数（Positional Arguments）准确的来说是位置参数，既然叫位置参数，那么参数就是和所处的位置有关，一般而言位置参数是被操作的主体对象，而且通常是必填的。

以 cp 命令举例，其操作格式是：

```
Usage: cp [OPTION]... [-T] SOURCE DEST
```

这里的 SOURCE 和 DEST 就是位置参数，cp foo bar 和 cp bar foo 是完全不同的，前者的source 是 foo，后者的 source 是 bar。

一般操作的主体可以是：文件、URL地址、软件包、镜像等等。

### 选项

选项（Option）是另外一种 CLI 的传参手段，和位置参数不同，它一般是用用连字符和单字母名称 (比如：-l、-h、-v) 或双连字符和多字母名称 (比如：--list、--help、--version）。

由于有了连字符后面的名称，它也可以看作是命名参数。命名参数的位置和顺序一般是不会影响程序的，命名参数的缺点是要多敲一些字符，但相比位置参数的优势也很明显，它更加清晰和人性化，位置参数需要记忆参数固定的位置，很容易写错顺序。

还是用 cp 举例，如果设计成位置参数：

```
cp /etc/myapp/config.ini /etc/myapp/config.ini.bak
```
这两个路径的顺序是不能调换的。但是，如果设计成选项（或者说是命名参数），需要多敲几下键盘，就不那么容易搞错顺序了：

```
cp --source /etc/myapp/config.ini --destination /etc/myapp/config.ini.bak

# 选项和顺序一般是无关的，所以也可以写成
cp --destination /etc/myapp/config.ini.bak --source /etc/myapp/config.ini
```

如果一个选项表示布尔值，那么这个选项就是一个标志位（Flag）或者说开关（Switch），比如 ls 的-l 参数或者 rm 的 -f 参数，就是标志位，它不接受参数，而是一个布尔值，默认不写为 False，反之为 True。

对于选项而言，有短选项和长选项，比如 -h 和 --help，在设计 CLI 的时候，仅仅需要为常用选项提供短选项，对于那些几乎不使用的选项仅提供长选项，因为对于大型的命令行工具而言，短选项只有 26 个英文字符的大小写选项，是有限的，要把有限的单字母留给那些最常用的选项。

### 子命令

子命令一般在比较大型的命令行工具中才会出现，它主要是用来切分业务逻辑，降低整体复杂性。

这里值得一提的是子命令的选项，对于每个子命令都有的选项（比如 -h），我们称其为全局选项，要保证全局选项在每一个子命令的一致性，例如 uv （Python的一个项目管理器）就明确提供了全局选项：

```shell
~$ uv
An extremely fast Python package manager.

Usage: uv [OPTIONS] <COMMAND>

# uv 的子命令
Commands:
  auth     Manage authentication
  run      Run a command or script
  init     Create a new project
  add      Add dependencies to the project
  remove   Remove dependencies from the project
  version  Read or update the project's version
  sync     Update the project's environment
  lock     Update the project's lockfile
  export   Export the project's lockfile to an alternate format
  tree     Display the project's dependency tree
  format   Format Python code in the project
  tool     Run and install commands provided by Python packages
  python   Manage Python versions and installations
  pip      Manage Python packages with a pip-compatible interface
  venv     Create a virtual environment
  build    Build Python packages into source distributions and wheels
  publish  Upload distributions to an index
  cache    Manage uv's cache
  self     Manage the uv executable
  help     Display documentation for a command

# 省略部分内容

# 全局选项，意味着每一个子命令都有该选项
Global options:
  -q, --quiet...                                   Use quiet output
  -v, --verbose...                                 Use verbose output
      --color <COLOR_CHOICE>                       Control the use of color in output [possible values: auto, always, never]
      --native-tls                                 Whether to load TLS certificates from the platform's native certificate store [env: UV_NATIVE_TLS=]
      --offline                                    Disable network access [env: UV_OFFLINE=]
      --allow-insecure-host <ALLOW_INSECURE_HOST>  Allow insecure connections to a host [env: UV_INSECURE_HOST=]
      --no-progress                                Hide all progress outputs [env: UV_NO_PROGRESS=]
      --directory <DIRECTORY>                      Change to the given directory prior to running the command [env: UV_WORKING_DIRECTORY=]
      --project <PROJECT>                          Run the command within the given project directory [env: UV_PROJECT=]
      --config-file <CONFIG_FILE>                  The path to a `uv.toml` file to use for configuration [env: UV_CONFIG_FILE=]
      --no-config                                  Avoid discovering configuration files (`pyproject.toml`, `uv.toml`) [env: UV_NO_CONFIG=]
  -h, --help                                       Display the concise help for this command
  -V, --version                                    Display the uv version

Use `uv help` for more details.

```
---

## 入乡随俗

就像现实社会中有很多不成文的规则一样，CLI 由于其文本属性导致用户对默认的规则非常依赖。几乎所有的 CLI 都会使用 --version 输出版本号，以及 --help 输出帮助文档，这就是约定俗成的选项。

以下还有些常见的选项，最好设计成用户习惯的语义：

- a, all: 全部
- d, debug: 输出 debug 级别的日志信息
- f, force: 强制执行某个命令，比如 rm 的 -f 选项
- u, user: 用户名
- p, port or password or preserve: 端口，密码，保留属性
- v, verbose or version: 详细输出，版本号
- o, output: 输出文件

这里有一些选项是有多个含义的，最容易搞错的就是这个 -p，有的用 -p 表示端口（psql，ssh，redis-cli），有的用 -P 表示端口（mysql，scp）。

这种情况下就要取决于开发 CLI 的程序员的选择，我个人更加倾向于 -p/--port 表示端口，--password 表示密码。


---

## 外貌主义


---

## 错误处理

---

## 远离等待


---

## 多些提示

---


## 拒绝歧义

---

## 再次确认


---

## 支持配置

---

## 防御输入


---





## 参考

1. https://sunbk201.github.io/cli-guidelines-zh/
2. https://clig.dev/
3. https://medium.com/@jdxcode/12-factor-cli-apps-dd3c227a0e46
4. https://typer.tiangolo.com/
5. https://zhuanlan.zhihu.com/p/593598678
6. https://news.ycombinator.com/item?id=25304257
7. https://blog.joway.io/posts/the-art-of-cli-design/
8. https://blog.atlan.com/engineering/the-art-of-building-delightful-clis-lessons-learned-from-building-the-atlan-cli/
9. https://xie.infoq.cn/article/3dbfeb01f181c3f0d1fc98862
10. https://unix.stackexchange.com/questions/285575/whats-the-difference-between-a-flag-an-option-and-an-argument
