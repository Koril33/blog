---
title: "PG的字符类型"
date: 2026-05-17T11:47:16
summary: "char varchar text 的区别"
---

## 两种字符串

SQL 标准定义了两种类型的字符串：

- character varying
- character

第一种的别名是 varchar，表示变长字符串，第二种的别名是 char，表示定长字符串。

二者都可以接受一个参数 n，在定长字符串 char 里，n 表示固定的长度，如果存储的字符串长度 < n，数据库会自动填充空格。在变长字符串里，n 表示最大的长度，两个类型如何碰到存储超过 n 的字符串，都会报错。

n 表示的字符数，而不是字节数。

n 的范围：必须大于 0 小于 10485760，如果 char 没有传入 n，等价于 char(1)，如果 varchar 没有传入 n，则等价于无限长度和 text 一样（并非无限，存储字符串长度不超过大约 1GB）。

---

## text

text 是 PG 内部原生的字符串类型，varchar 更像是一种 text 的变体，PG 底层函数使用的都是 text 类型。

---

## 使用哪一个？

文档的 Tip 写道：这三种类型之间没有性能差异,除了使用空白填充类型时增加了存储空间,以及在将长度限制的列存储时,会多检查几个CPU周期。尽管 char(n) 在某些其他数据库系统中具有性能优势,但在PostgreSQL中并不存在此类优势;事实上,由于额外的存储成本, char(n) 通常是三者中最慢的。在大多数情况下,应改用 text 或 varchar 的格式。

---

## 要不要指定 n

文档写道：If you desire to store long strings with no specific upper limit, use text or character varying without a length specifier, rather than making up an arbitrary length limit.)

所以，PG 里的字符串其实不用需要考虑性能，如果没有明确的限制，直接使用不指定 n 的 varchar 或者 text，除非明确需要数据库限制长度，使用 varchar(n)，或者明确一个字段是固定长度的（几乎不会更改，比如邮编，车牌号，证件号），则最后再考虑使用char(n)。

text 是 PG 的内置类型，支持很好，varchar 曾更像是一种带有限制的 text，而 char 的性能是三者里最差的，所以优先级如下：

text > varchar > varchar(n) > char

不需要随意乱写一个 varchar(255)，直接使用 text 或者不要指定 n 的 varchar 就行。

---

## 参考

1. https://www.postgresql.org/docs/17/datatype-character.html
