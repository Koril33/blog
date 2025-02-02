---
title: "Debian12安装Redis"
date: 2024-11-17T16:25:48+08:00
tags: []
featured_image: "images/background.jpg"
summary: "安装 Redis 以及设置密码开放远程连接"
toc: true
---

## 前言

本文介绍了如何在 Debian 12 上安装和启用 Redis 数据库服务器的访问控制。

---

## 步骤

### 更新 apt

```bash
sudo apt update && sudo apt upgrade -y
```

### 查看是否已经安装了 Redis

```bash
dpkg -l | grep redis
```

### 安装 Redis

```bash
sudo apt install redis
```

### 查看安装版本

```sh
redis-server -v
```

### Start 和 Enable Redis

安装成功后，redis 默认是自动启动，可以检查下：

```bash
sudo systemctl status redis
```

如果没有启动，可以手动启动：

```bash
sudo systemctl start redis
```

开机自启：

```bash
sudo systemctl enable redis
```

---

### 开放远程连接

redis 默认只允许本地连接，开放远程连接，需要修改 /etc/redis/redis.conf

```sh
# 备份
sudo cp /etc/redis/redis.conf /etc/redis/redis.conf.bak

# 修改
sudo vim /etc/redis/redis.conf

# 找到 bind 这一行
# 原本是 bind 127.0.0.1 -::1
# 改成
bind 0.0.0.0
```

### 设置密码

开放远程连接后，如果不设置密码，还是没法远程连接，因为有个 redis.conf 有个参数是：

```
protected-mode yes
```

除非关闭（改成 no），否则仍然只能 localhost 访问，所以必须要设置密码。

修改 /etc/redis/redis.conf

```sh
# 修改
sudo vim /etc/redis/redis.conf

# 找到 requirepass 这一行，然后添加密码
# requirepass foobar
# 改成自己的密码
requirepass 123456
```

修改完配置文件，重启 redis

```sh
sudo systemctl restart redis
```

---

## 参考

1. [How to Install Redis® on Debian 12 | Vultr Docs](https://docs.vultr.com/how-to-install-redis-on-debian-12)
