---
title: "ssh心跳保活"
date: 2025-02-06T20:03:00+08:00
summary: "解决 ssh 连接会断开的问题"
---

## 目录

[TOC]

---

## 前言

使用 OpenSSH 客户端连接服务器的时候，发现超过一定的时间如果不操作，SSH 连接会自动断开。

---

## 解决方案

OpenSSH 提供了心跳保活的功能，可以在服务端设置，也可以在客户端设置。

### 服务端设置

在`/etc/ssh/sshd_config`中添加下面两行：

```
ClientAliveInterval 60
ClientAliveCountMax 3
```

服务器每 60 秒 发送一个心跳包（ClientAlive）。
如果客户端连续 3 次 没有响应（即 180 秒），服务器才会主动断开连接。

### 客户端设置

大多数情况下，我们没有修改服务器 sshd 的权限，在客户端设置心跳发送，也可以达到保活的效果。

如果你只是想防止 ssh 连接超时，建议在 客户端 的 ~/.ssh/config 里加：

```
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

这样每 60 秒客户端向服务器发送一次心跳，最多允许 3 次 无响应，总超时时间约 180 秒。

## 服务端 or 客户端？

客户端设置 (ServerAliveInterval) 更适合主动维护连接，从家里 ssh 登录远程服务器，希望防止 NAT/防火墙导致的连接断开。适用于无权限修改服务器配置的情况。

服务器设置 (ClientAliveInterval) 更适合检测客户端是否存活，服务器上 sshd 需要及时清理失效的 ssh 连接（防止长期占用资源）。适用于管理员有权限修改服务器配置的情况。
