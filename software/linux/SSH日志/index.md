---
title: "SSH日志"
date: 2025-06-29T10:00:00+08:00
summary: "如何在 Debian 12 下查看 SSH Server 的日志信息"
---

## 目录

[TOC]

---

## 前言

公网服务器的 SSH 服务往往是黑客重点关照对象，本文简单的介绍下 Debian12 下如何查看 SSH Server 的日志信息。

---

## 查看日志

Debian12 默认使用了 systemctl 管理 sshd 服务，所以日志需要通过 journalctl 查看。

查看 ssh 日志：

```sh
sudo journalctl -u ssh
```

这样会按照时间顺序（从最早开始）显示 ssh 的日志信息。如果想要从最晚时间开始显示（倒序），指定 -r 参数：

```sh
sudo journalctl -u ssh -r
```

journalctl 支持 -f 参数，效果和 tail -f 类似，默认显示最近 10 条（-n 参数可以修改这个数量）日志：

```sh
sudo journalctl -u ssh -f
```
---

## 日志内容

实际的日志类似如下：

```
Jun 29 08:52:59 my-debian sshd[179811]: Failed password for root from 139.128.106.99 port 29699 ssh2
```

包含时间辍，服务器名称（hostname），进程名字，进程 PID，冒号后面就是日志信息，对于 SSH 而言，主要是身份验证是否通过。

成功信息：

- 通过用户名密码登陆成功：Accepted password for root from 139.188.102.101 port 29806 ssh2
- 通过公私钥登陆：Accepted publickey for root from 139.208.16.19 port 29784 ssh2: ED25519 SHA256:qxxaRehIr1...

失败信息：

- 使用密码尝试登录，但密码错误：Failed password for root from 121.37.128.117 port 55686 ssh2
- 试图使用系统中不存在的用户：Failed password for invalid user admin from 222.175.39.226 port 54526 ssh2
- 客户端尝试了太多认证方式，SSH 服务主动断开：Connection closed by authenticating user root 121.37.128.117 port 55686 [preauth]

失败信息还有很多，不一一列举。

---

## 检索

检索常用的关键字如下：

```sh
# 查看所有失败记录（密码失败、无效用户等）
journalctl -u ssh | grep "Failed"

# 查看无效用户尝试
journalctl -u ssh | grep "invalid user"

# 使用公钥登陆记录
journalctl -u ssh | grep "publickey"

# 查看所有尝试记录（成功 + 失败）
journalctl -u ssh | grep -E "Accepted|Failed"
```

---

## 参考

1. https://wiki.debian.org/SSH
2. https://www.strongdm.com/blog/view-ssh-logs
3. https://www.jinbuguo.com/systemd/journalctl.html
4. https://last9.io/blog/sshd-logs-101/
