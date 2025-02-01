---
title: "Linux之间使用scp命令传输文件"
date: 2023-05-07T11:33:54+08:00
tags: []
featured_image: "images/background.jpg"
summary: "介绍一下在两台Linux主机之间如何通过scp互相传输文件"
toc: true
---

## 前言

在服务器的运维中，文件传输是一个常常用得到的功能，本地文件的拷贝可以使用`cp`命令，而不同主机的文件拷贝则可以使用`scp`命令。

---

## SCP

SCP 的全称是 Secure Copy，意为安全拷贝，它使用 SSH 作为协议基础，在 SSH 之上实现了文件的远程拷贝传输。

通过`scp`命令传输的文件是经过加密的，所以`scp`是一个安全的传输命令。

使用 SCP 可以从本地传输文件到远程主机（上传），也可以从远程主机传输文件到本地（下载）。

### SCP 语法

下面是 SCP 的语法结构：

```shell
scp [FLAG] [user@]SOURCE_HOST:]/path/to/file1 [user@]DESTINATION_HOST:]/path/to/file2
```

`[FLAG]`指定可以提供给 SCP 的选项。以下是有关标志的一些详细信息：

| FLAG | DESCRIPTION                              |
| :--- | :--------------------------------------- |
| -r   | 递归拷贝目录                             |
| -q   | 用于隐藏下载进度和除了错误之外的一切信息 |
| -C   | 用于将数据传输到目的地时压缩数据         |
| -P   | 指定目的地的SSH端口                      |
| -p   | 保留文件访问时间                         |

`[user@]SOURCE_HOST:]`是源主机

`[user@]DESTINATION_HOST:]`是目标主机

注意：要通过 SCP 传输文档，必须知道凭据（比如密码），并且用户应具有写入权限。

现在，我以两台局域网内的 Ubuntu 机器作为举例，一个 IP 为 192.168.10.106，一个 IP 为 192.168.10.107。

### 上传本地文件到远程主机

登录 106 机器，并且新建一个文本文件——hello.txt，向里面随便写入一些内容：

```shell
ssh ubuntu@192.168.10.106
echo "hello scp!" > hello.txt
```

将 hello.txt 传输到 107 的机器上的 /home/ubuntu 目录下：

```shell
scp hello.txt ubuntu@192.168.10.107:/home/ubuntu
```

输入 107 机器的凭证，传输成功：

```shell
ubuntu@192.168.10.107's password:
hello.txt                                                                             100%   11     3.0KB/s   00:00
```

如果需要传输目录，需要加上 -r 参数，递归拷贝，比如这里将 hello.txt 移动到 test 目录下，再传输 test 目录：

```shell
mkdir test
mv hello.txt test/
scp -r test/ ubuntu@192.168.10.107:/home/ubuntu
```

### 下载远程主机文件到本地

和上面的命令相似，只不过把本地和远程调换一下：

```shell
scp ubuntu@192.168.10.107:/home/ubuntu/107.txt /home/ubuntu
```

同样的，传输目录，加上 -r 参数：

```shell
scp -r ubuntu@192.168.10.107:/home/ubuntu/test /home/ubuntu
```

### 不同的 SSH 端口

上面的命令都默认了目标的主机使用的是 22 端口作为 SSH 端口，在实际操作中，我们会将 22 端口改成别的端口，则需要增加 -P 参数，比如我将 107 的 ssh 端口改成了 22222：

```shell
scp -P 22222 -r test/ ubuntu@198.168.10.107:/home/ubuntu
```

---

## 参考

1. https://www.freecodecamp.org/news/how-to-transfer-files-between-servers-in-linux-using-scp-and-ftp/
2. https://wangchujiang.com/linux-command/c/scp.html
3. https://help.ubuntu.com/community/SSH/TransferFiles
