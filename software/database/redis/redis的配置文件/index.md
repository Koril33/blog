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

在生产环境下，对于业务体量的预估是很重要的。

生产服务器上的某一个进程在磁盘或者内存的使用没有限制时，往往它就会变成一个不知道什么时候会爆炸的隐形炸药包。

例如：对于日志文件，不进行大小限制或者轮转，极有可能把磁盘撑爆了，同样对于 redis 这种内存数据库而言，在生产环境下，预估好体量，然后限制 redis 最大的内存使用量，可以避免内存被撑爆的风险。

配置内存占用的最大值的配置名称是`maxmemory`，默认没有指定，在 32 位环境下，不指定 maxmemory 最大内存占用为 3GB（redis 代码层面限制），而 64 位环境下，不指定 maxmemory 则不进行内存大小限制。

假设想要限制一个 redis-server 最大内存占用，指定以下配置：

```
# 最大 10 MB
maxmemory 10mb
```

内存单位有多种写法，参考 redis.conf 开头的注释：

```
# Note on units: when memory size is needed, it is possible to specify
# it in the usual form of 1k 5GB 4M and so forth:
#
# 1k => 1000 bytes
# 1kb => 1024 bytes
# 1m => 1000000 bytes
# 1mb => 1024*1024 bytes
# 1g => 1000000000 bytes
# 1gb => 1024*1024*1024 bytes
#
# units are case insensitive so 1GB 1Gb 1gB are all the same.
```

指定好后可以写一个简单的脚本测试，到达了内存设置最大值会发生什么？

```python
def main():
    redis_client = db.get_redis_client()
    for i in range(2000_000):
        redis_client.lpush('tasks', i)
```

结果是爆出以下异常：

```
redis.exceptions.OutOfMemoryError: command not allowed when used memory > 'maxmemory'.
```

这里引出了第二个配置项，maxmemory-policy，表示当达到最大内存限制时，Redis 将如何选择要移除的内容。

redis 提供了以下几个策略：

1. volatile-lru -> 使用近似 LRU 算法逐出，仅移除设置了过期时间的键。
2. allkeys-lru -> 使用近似 LRU 算法逐出任何键。
3. volatile-lfu -> 使用近似 LFU 算法逐出，仅移除设置了过期时间的键。
4. allkeys-lfu -> 使用近似 LFU 算法逐出任何键。
5. volatile-random -> 随机移除一个设置了过期时间的键。
6. allkeys-random -> 随机移除一个键，任意键。
7. volatile-ttl -> 移除过期时间最接近的键（minor TTL）。
8. noeviction -> 不逐出任何键，仅在写入操作时返回错误。

LRU 表示最近最少使用
LFU 表示最不频繁使用

LRU、LFU 和 volatile-TTL 均采用近似随机算法实现。

maxmemory-policy 的默认值是 noeviction，所以刚刚的测试脚本超过了最大值，直接返回错误。

---

## 参考

1. https://redis.io/docs/latest/operate/oss_and_stack/management/config/
1. https://redis.com.cn/redis-configuration.html
1. https://dev.to/rijultp/understanding-redisconf-how-to-configure-your-redis-server-38lf
