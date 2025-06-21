---
title: "redis的配置文件"
date: 2025-06-21T11:13:00+08:00
summary: "redis 配置文件的各个字段释义"
---

## 目录

[TOC]

---

## 配置的类型

redis 没有指定任何配置文件的时候，会使用内置的配置文件。

配置可以通过命令行提供。

在运行时也可以通过 CONFIG SET 指令修改配置，但是这种动态配置不会影响 redis.conf，下次重启后还是使用原先的配置文件信息。

---

## 配置文件

配置文件一般存放在 `/etc/redis/redis.conf`

每一个版本的配置文件原始文本，都可以在官网找到，比如 7.0 的配置文件：https://raw.githubusercontent.com/redis/redis/7.0/redis.conf

接下来，按照最常用最重要的配置，依次讲解。

### 绑定 IP 和端口

默认配置下，redis 仅允许本地网络（127.0.0.1）访问 redis-server，端口默认是 6379，可以通过以下配置修改：

```
bind 0.0.0.0
port 16379
```

以上修改，表示 redis 接受所有远程连接，并且端口修改成了 16379

### 保护模式

redis 为了避免用户在没有设置密码的情况下，把 redis-server 暴露在公网上（比如上面的 bind 0.0.0.0），默认提供了一个配置参数——protected-mode，并且默认是 yes，表示启用保护模式。

它的效果是，如果启用保护模式，并且默认用户没有密码时，redis-server 仅接受来自 IPv4 地址 (127.0.0.1)、IPv6 地址或 Unix 域套接字的本地连接。

我们当然也可以把 protected-mode 设置为 no，但是与此同时，一定要设置个密码，不然所有人都可以访问 redis-server 了。

### 用户和密码

Redis 6.0 之前没有用户的概念，只有一个密码，就可以访问 redis-server 的所有内容，Redis 6.0 以后新增了 ACL（Access Control List）的功能，可以指定多个不同的用户名和密码。

为了向后兼容性，Redis 6.0 默认有个名为 default 的账号，我们在配置文件里通过参数 requirepass 设置的密码，实际上就是给 default 设置密码：

```
requirepass foobared
```

### 快照



### 内存控制







---

## 参考

1. https://redis.io/docs/latest/operate/oss_and_stack/management/config/
1. https://redis.com.cn/redis-configuration.html
1. https://dev.to/rijultp/understanding-redisconf-how-to-configure-your-redis-server-38lf