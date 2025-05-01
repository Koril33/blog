---
title: "Debian12安装jdk21"
date: 2024-10-20T10:09:00+08:00
tags: []
featured_image: "images/background.jpg"
summary: "下载压缩包/配置JDK21环境变量"
toc: true
---

## 前言

在 Debian 上 安装 jdk21 一般有三种方式：

1. apt install
2. 下载 deb 包，dpkg 安装
3. 下载 tar.gz 包，解压安装

前两种很简单，本文仅记录第三种方式。

---

## 安装步骤

### 下载 jdk21 压缩包

地址：

[Java Downloads | Oracle India](https://www.oracle.com/in/java/technologies/downloads/#java21)

我的机器和系统是 X64/Debian12，所以选择 x64 Compressed Archive 这个选项，本地下载好后，上传到目标机器里，目标机器如果联网了，也可以在目标机器中用 curl 或 wget 下载压缩包。

### 解压缩

```bash
tar -xzvf jdk-21_linux-x64_bin.tar.gz
```

### 创建目录

```bash
sudo mkdir /opt/java
sudo mv jdk-21.0.5/ /opt/java/
```

### 配置环境变量

```bash
sudo vim /etc/profile.d/jdk21.sh
```

内容如下：

```bash
export JAVA_HOME="/opt/java/jdk-21.0.5"
export PATH="$PATH:$JAVA_HOME/bin"
```

### source 配置文件

```bash
source /etc/profile.d/jdk21.sh
```

### 检查

```bash
java -version

# 输出
java version "21.0.5" 2024-10-15 LTS
Java(TM) SE Runtime Environment (build 21.0.5+9-LTS-239)
Java HotSpot(TM) 64-Bit Server VM (build 21.0.5+9-LTS-239, mixed mode, sharing)
```

