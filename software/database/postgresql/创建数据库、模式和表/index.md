---
title: "创建数据库、模式和表"
date: 2026-05-02T20:49:38
summary: ""
---

## 目录

[TOC]

---

## 前言

PG 对于 SQL 标准的实现很严谨，在组织数据的层面上，用三层概念来分隔数据：

1. database：物理层面的隔离
2. schema：在逻辑上进行隔离，提供一种命名空间的机制
3. table：真正存储数据的容器

本文介绍如何创建、修改以及删除数据库、模式和表。

---

## 查看

为了方便后续查看信息，这里先给出查看信息的方式

### psql

查看数据库：

```
\l
or
\list
```

详细信息：

```
\l+
```

查看某个数据库：

```
\l postgres
```

查看模式：

```
\dn
```

查看表：

```
\dt 
```

### 系统表

查看数据库：

```sql
SELECT * FROM pg_database;
```

查看模式：

```sql
SELECT * from pg_namespace;
```

---

## 创建

### 数据库

创建一个数据库很简单：

```sql
CREATE DATABASE study;
```
数据库名称必须唯一，不然会爆出 `ERROR:  database "study" already exists`，稍微复杂一些的，是各种配置项。

首先是数据库的所有者（Owner），我们在创建数据库的时候可以用以下指令给数据库分配一个 Owner：

```sql
CREATE DATABASE study OWNER koril;
```

如果创建时没有指定 Owner，默认这个数据库的所有者就是执行`CREATE DATABASE`指令的角色。

默认创建数据库，没有限制连接数，连接数都是`-1`，如果想要限制新建数据库的最大连接数，使用`CONNECTION LIMIT`：

```sql
CREATE DATABASE study CONNECTION LIMIT 10;
```

完整的参数列表如下：

```sql
CREATE DATABASE database_name
WITH
   [OWNER =  role_name]
   [TEMPLATE = template]
   [ENCODING = encoding]
   [LC_COLLATE = collate]
   [LC_CTYPE = ctype]
   [TABLESPACE = tablespace_name]
   [ALLOW_CONNECTIONS = true | false]
   [CONNECTION LIMIT = max_concurrent_connection]
   [IS_TEMPLATE = true | false ];
```

### 模式

数据库提供了物理层面的隔离，那么模式（Schema）就是逻辑层面的隔离，提供了一种类似命名空间（Namespace）的机制。

一个数据库可以包含多个模式，一个模式包含表，以及各种命名对象，包括：数据类型（data type）、函数（function）、视图（view）、索引（index）。

在同一个模式下，对象需要具有不同的名字，比如 index 和 table 不能重名，但是在不同模式下，一个名字可以重复使用，比如多租户系统下，schema-1 包含了 user 表，schema-2 也可以包含一个同样名字的 user 表。

在以下情况时可以考虑使用多个schema来管理数据库：

1. 多个用户使用同一个数据库，每个用户使用一个 schema，互不影响。
2. 一个项目下的不同子项目，比如微服务，可以使用多个 schema。

每一个数据库新建后，PG 都会自动创建一个名为 public 的 schema，如果创建表或者其他对象时，不制定 schema，那么这些对象默认归属于 public 模式。

public 模式和自建的模式没有区别，但是它有些默认的设置：

- 创建 database 时，会自动创建 public 模式
- 默认在 search path 中
- 默认所有人都有 public 的 USAGE 权限（PG 14 以前是 CREATE 和 USAGE, PG 15 以后仅有 USAGE）


创建新的 schema：

```sql
CREATE SCHEMA sale;
```

授权：

```sql
CREATE SCHEMA sale AUTHORIZATION alice;
```

---

## search path

大部分时候，在执行 SQL 里对于表的引用往往忽略前缀的 schema，是因为 PG 有一套类似于 Java/Python 的 class path。PG 会自动从 schema search path 里面找，定位到第一个匹配的结果。如果在所有 search path 都没有找到用户指定的表名，就会报出 relation 不存在错误。

查看 search path：

```sql
SHOW search_path;
```

默认情况下返回：

```
"$user", public
```

含义是：

1. 先找一个和当前用户同名的 schema
2. 如果没有，就用 public

以下命令查看当前默认 schema

```sql
SELECT current_schema();
```

`current_schema()`会返回当前`search_path`中“第一个存在且可用的 schema”。

假设 search path 有 a b c 三个schema,但是 a 和 b 都不存在或者不可访问（权限），c 存在且可以访问，那么`current_schema`就会返回 c。

创建表的时候，不指定 schema，PG 运行的原理大致可以粗略的理解成：

```
CREATE TABLE test(id int);

CREATE TABLE current_schema().test;
```

查询表的时候，不制定 schema，则可以理解为：

```
SELECT * FROM orders;

for schema in search_path:
  if table exists in schema:
    use table
    break
```

---


## 修改



---

## 删除


---

## 实践

### 数据库和模式的区别


### 权限原则


---

## 参考

1. https://neon.com/postgresql/administration/create-database
2. https://neon.com/postgresql/administration/create-database
