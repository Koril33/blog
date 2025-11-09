---
title: "uv设置镜像源"
date: 2025-10-25T12:00:00+08:00
summary: "又到了纠结如何设置镜像源的时候"
---

## 目录

[TOC]

---

## 前言

和 pip 类似，uv 也是从 pypi 下载包，但是有时候国内无法下载，就需要设置镜像源。

---

## 临时使用

通过 export 声明环境变量:

```
export UV_DEFAULT_INDEX=镜像源地址
```

或者在命令行使用 --default-index:

```
uv add requests --default-index 镜像源地址
```

---

## 永久配置

可以把上面提到的 UV\_DEFUALT\_INDEX 写进 ~/.profile（或 ~/.bashrc、~/.zshrc）里。

或者通过配置文件的方式，uv 会在当前项目或全局配置中读取 uv.toml。
你可以在其中定义默认索引、额外索引、缓存策略等。

全局配置地址：~/.config/uv/uv.toml

写入内容如下：

```toml
[[index]]
url = "https://pypi.mirrors.ustc.edu.cn/simple/"
default = true
```

---

## 参考

1. https://docs.astral.sh/uv/concepts/indexes/#defining-an-index
