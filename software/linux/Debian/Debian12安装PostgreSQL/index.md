---
title: "Debian12安装PostgreSQL"
date: 2024-11-10T09:04:00+08:00
tags: []
featured_image: "images/background.jpg"
summary: "安装 PostgreSQL 以及设置密码开放远程连接"
toc: true
---

## 前言

PostgreSQL 是一种开源关系数据库管理系统 （RDBMS），支持结构化查询语言 （SQL）。

数据库服务器适用于在各种信息系统中存储数据和运行查询。PostgreSQL 最显著的性能特性包括多版本并发控制、事务支持、外键、用户定义的数据类型和自定义视图。

本文介绍了如何在 Debian 12 上安装和启用 PostgreSQL 数据库服务器的访问控制。

---

## 步骤

### 更新 apt

```bash
sudo apt update && sudo apt upgrade -y
```

### 查看是否已经安装了 PostgreSQL

```bash
dpkg -l | grep postgresql
```

这个命令会列出所有包含 `postgresql` 的已安装包，如果看到 `postgresql` 和 `postgresql-contrib` 出现在列表中，说明它们已经安装了。

### 安装 PostgreSQL

```bash
sudo apt install postgresql postgresql-contrib
```

postgresql：PostgreSQL 数据库管理系统的主包，包含核心的数据库服务器和管理工具。安装这个包会提供 PostgreSQL 的基本功能，能够创建和管理数据库、处理 SQL 查询等。

postgresql-contrib：包含一些 PostgreSQL 提供的额外扩展和模块，这些功能不是 PostgreSQL 核心的一部分，但可以增强数据库的功能和灵活性。

### Start 和 Enable PostgreSQL

安装成功后，PostgreSQL 默认是自动启动，可以检查下：

```bash
sudo systemctl status postgresql
```

如果没有启动，可以手动启动：

```bash
sudo systemctl start postgresql
```

开机自启：

```bash
sudo systemctl enable postgresql
```

---

### 用户、密码

默认情况下，PostgreSQL 会创建一个名为“postgres”的用户，该用户对整个 PostgreSQL 实例具有完全管理访问权限。出于安全考虑，建议为该用户设置密码。切换到“postgres”用户并访问 PostgreSQL 提示符：

```bash
sudo -i -u postgres
```

* sudo：以超级用户（root）权限执行命令。
* -i：以登录 shell 的方式启动一个会话。这样会加载用户的环境变量和配置（如 .bashrc、.profile 等）。
* -u postgres：指定要切换的用户。在 PostgreSQL 的默认安装中，会创建一个名为 postgres 的系统用户，用于管理数据库服务器。

这个命令的效果是让你以 postgres 用户的身份登录，以便直接执行数据库管理命令（如 psql）或其他与 PostgreSQL 相关的操作。

进入 PostgreSQL 的命令行：

```bash
psql
```

psql 是 PostgreSQL 的交互式终端。

为默认的 postgres 用户设置密码：

```bash
\password postgres
```

### 开放远程连接

默认 PostgreSQL 只允许本地连接，即 localhost，如果想要远程应用能连接，需要配置文件做些调整。

需要配置两个文件：

1. postgresql.conf
2. pg_hba.conf

在 Debian 12 上，它们位于 `/etc/postgresql/15/main/` 目录下。

备份初始文件：

```bash
sudo cp pg_hba.conf pg_hba.conf.bak
sudo cp postgresql.conf postgresql.conf.bak
```

在 `postgresql.conf` 中，将 `listen_addresses` 参数修改为 `'*'`，允许数据库服务器监听所有 IP 地址：

```bash
listen_addresses = '*'
```

在 pg_hba.conf 文件中添加一条规则，允许特定 IP 地址段的用户远程连接。例如，允许 192.168.1.0/24 网段的用户使用用户名和密码进行连接，可以添加以下内容：

```bash
host    all             all             192.168.1.0/24            md5
```

这行配置的含义是：

- `host`：表示允许通过 TCP/IP 连接。
- `all`：适用于所有数据库。
- `all`：适用于所有用户。
- `192.168.1.0/24`：指定允许访问的 IP 段。
- `md5`：使用 MD5 密码认证方式（也可以选择 `scram-sha-256` 等其他方式）。

如果想要所有 IP 都能访问（比如测试环境），可以添加以下内容：

```bash
host    all             all              0.0.0.0/0                       md5
```

修改完配置文件后，不要忘记重启 postgresql，让配置生效：

```bash
sudo systemctl restart postgresql
```



---

## 参考

1. [PostgreSQL: Linux downloads (Debian)](https://www.postgresql.org/download/linux/debian/)
2. [Installing PostgreSQL on Debian 12 for Beginners | Reintech media](https://reintech.io/blog/installing-postgresql-on-debian-12-for-beginners)
3. [windows - How to Allow Remote Access to PostgreSQL database - Stack Overflow](https://stackoverflow.com/questions/18580066/how-to-allow-remote-access-to-postgresql-database)
