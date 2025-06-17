---
title: "sqlite的数据类型"
date: 2025-06-02T11:05:00+08:00
summary: "sqlite 各种数据类型的介绍"
---

## 目录

[TOC]

---

## 前言

sqlite 的数据类型相对于其他的关系型数据库而言，最大的特点在于其动态性。其他的关系型数据库的数据类型是强类型，数据类型由其容器决定（也就是在定义列的时候），而 sqlite 的数据类型则和值本身关联。

这种灵活的数据类型是 sqlite 的一个特性，而非 bug（类似于静态类型语言和动态类型语言的区别）。

---

## 存储类型

每一个存储在 sqlite 中的值，都属于下面几个类型中的一个：

* NULL，空
* INTEGER，存储 intergere，根据值的大小存储在 0/1/2/3/4/6/8 个 bytes 中
* REAL，浮点数
* TEXT，字符串，文本类型，根据数据库配置进行编码（UTF-8/UTF-16BE/UTF-16LE）
* BLOB，二进制数据，存储的内容不进行额外的编码

在 SQL 语句中的值，会在执行期间进行类型转换（比如 integer 和 text），除了 integer primary key 定义的列之外，任何列都可以存储任何类别的值。

SQLite 对字符串、BLOB 或数值的长度没有施加任何限制，所以 varchar(50) 的 50 在 sqlite 中会被忽略。

---

## 布尔值

sqlite 没有单独的布尔类型，所以存储布尔值需要用 integer （0-false，1-true）来表示。

从版本 3.23.0（2018-04-02）开始，SQLite 可以识别关键字“TRUE”和“FALSE”，但这些关键字实际上只是 integer 1 和 0 的替代拼写。

---

## 时间

sqlite 没有单独的时间类型，sqlite 内置的日期及时间函数会将日期时间存储为 TEXT，REAL 或 INTEGER。

* Integer 存储 Unix 时间，1970-01-01 00:00:00 UTC 至今的秒数
* real 存储儒略日数
* text 存储 ISO-8601 字符串（YYYY-MM-DD HH:mm:ss.SSS）

应用程序可以选择以上述任意格式存储日期和时间，并使用内置的日期和时间函数自由地在格式之间进行转换。

---

## 类型亲和性

由于 sqlite 的类型很少（相比 mysql/pg 的 small int，big int，char，varchar），所以有类型亲和性的概念，这个特性我不是很喜欢，因为将数据库对于数据类型错误的包容性太大，对于类型的约束转移到了应用代码层面。

举个简单的例子：sqlite 的一列数据是可以存储不同类型的值的，这很灵活，一个定义为 TEXT 的列，存储 integer，real，blob 都可以。

这一块的内容，暂略。

---

## 参考

1. https://www.sqlite.org/datatype3.html
2. https://www.sqlite.org/flextypegood.html
3. https://wkbse.com/2025/04/03/sqlite-%e6%95%b0%e6%8d%ae%e7%b1%bb%e5%9e%8b%e4%ba%b2%e5%92%8c%e6%80%a7-affinity-%e6%b7%b1%e5%ba%a6%e8%a7%a3%e6%9e%90-wiki%e5%9f%ba%e5%9c%b0/