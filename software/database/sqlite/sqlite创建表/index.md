---
title: "sqlite创建表"
date: 2025-06-02T11:05:00+08:00
summary: "sqlite 创建表的语法"
---

## 目录

[TOC]

---

## 前言

本文简单介绍 sqlite 创建表的语法

---

## CREATE TABLE 相关参数

CREATE TABLE 可以指定以下的参数，来创建一张新表：

- 新表的名字
- 指定在哪一个数据库（模式名称 schema name）中，创建新表
- 新表的每一列的名称
- 每一列的数据类型
- 每一列的默认值或者表达式
- 每一列的排序规则
- PRIMARY KEY，主键，支持单列或者复合列（可选）
- 新表的 SQL 约束：UNIQUE、NOT NULL、CHECK、FOREIGN KEY
- 生成列（generated column）的约束（可选）
- 该表是否为 WITHOUT ROWID 表
- 表是否经过严格的类型检查。

注意：
1. 新表的名字不能以 sqlite_ 开头
2. 如果指定了模式名称，它必须是 main、temp 或者附加数据库名称中的一个
3. 如果该数据库中已存在一个表，再创建一个新表和已存在表重名的话，会报错，除非指定 IF NOT EXISTS 子句，指定了该子句，碰到同名表，不会执行创建表操作也不会报错。

语法：

```
CREATE TABLE [IF NOT EXISTS] [schema_name].table_name (
	column_1 data_type PRIMARY KEY,
   	column_2 data_type NOT NULL,
	column_3 data_type DEFAULT 0,
	table_constraints
) [WITHOUT ROWID];
```

对于列的定义，只有列名是必填项，也就是说下面的语句是合法的：

```
CREATE TABLE t_user (id, name);
```

因为 sqlite 的类型系统是宽松的动态类型，上面这张表不具备数据类型约束或其他完整性规则。
当然，一般都会加上数据类型，增加可读性和可维护性：

```
CREATE TABLE t_user (id INTEGER, name TEXT);
```


对于约束，有两种：列约束和表约束，这里先介绍列约束，sqlite 支持的约束如下：

PRIMARY KEY：主键约束，唯一且非空
UNIQUE：值唯一
NOT NULL：禁止列值为 NULL
DEFAULT：设置默认值
CHECK：自定义逻辑限制
FOREIGN KEY：外键约束，sqlite 的外键约束是默认关闭的，需要额外的命令手动开启

以下用一个简单的示例，覆盖以上所说的约束类型。
创建一个新表 t_user，主键是 id，用户姓名非空，年龄的值要比 0 大，email 值唯一，薪水默认为 0，创建时间默认为当前日期时间，用户地址为字符串。

```sql
CREATE TABLE t_user (
	id              INTEGER PRIMARY KEY,
	name            TEXT    NOT NULL,
	age             INTEGER CHECK(age >= 0),
	email           TEXT    UNIQUE,
	salary          REAL    DEFAULT 1000.5,
	create_datetime TEXT    DEFAULT CURRENT_TIMESTAMP,
    address         TEXT
);
```

### 主键自增

sqlite 的主键自增有两种：

1. INTEGER PRIMARY KEY
2. INTEGER PRIMARY KEY AUTOINCREMENT

如果 id 设置成了 INTEGER PRIMARY KEY 的话，sqlite 默认该字段自增，当 NULL 插入表的该列时，NULL 会自动转换为一个整数，该整数比该列的最大值大 1。

新键在当前表中的所有键中都是唯一的，但 id 可能与先前已从表中删除的键重叠。

这里就要提到 AUTOINCREMENT，如果声明了 AUTOINCREMENT，则表示强制自增，并且**新的键的值不会与之前删除的键重叠**。

另外，如果有一张表的主键设置了 AUTOINCREMENT，那么 sqlite 就会自动创建一个名为 sqlite_sequence 的系统表，用来记录每一个带有 AUTOINCREMENT 的表当前的自增序列数值：

```sql
CREATE TABLE sqlite_sequence(
  name TEXT,
  seq INTEGER
);
```

每新增一个带有 AUTOINCREMENT 的表，sqlite_sequence 就会多一条记录，删除了带有 AUTOINCREMENT 的表，sqlite_sequence 对应的那一条记录也会自动被删除。

### 约束

在 t_user 表中，除了 address 之外，其他字段都有约束，address 没有指定 NOT NULL，不插入 address 的话，address 默认值就是 NULL。其他字段有 DEFAUTL，不传的话，默认插入 DEFAULT 指定的值。
违反任何约束，都无法插入成功。

---

## 参考

1. https://sqlite.org/lang_createtable.html
2. https://sqlite.readdevdocs.com/lang_createtable.html
3. https://www.sqlitetutorial.net/sqlite-create-table/