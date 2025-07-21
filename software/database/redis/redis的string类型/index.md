---
title: "redis的配置文件"
date: 2025-06-21T11:13:00+08:00
summary: "redis 配置文件的各个字段释义"
---

## 目录

[TOC]

---

## string

string 是 redis 最基本的类型，一个 key 对应一个 value。

key 是 string，这里的 value 也是 string。

redis 的 string 是一种二进制安全的数据结构。字符串是字节（或单词）的数组数据结构，它使用一些字符编码存储一系列元素，通常是字符。它可以存储任何数据 - 字符串、整数、浮点值、JPEG 图像、序列化的 Ruby 对象或您希望它携带的任何其他数据。

---

## 二进制安全

二进制安全（binary-safe） 意味着 redis 存储和操作字符串时，完全按“字节流”处理，而不会尝试解析、解释或修改它的内容。二进制的内容，怎么存入到 redis 的，就原模原样取出来。

这让你可以放心地把任何数据（文本、图像、压缩包、序列化对象）都当作字符串存入 redis，而不会出问题。

C 语言的字符串就不是二进制安全的，因为它会把 '\0' 作为特殊含义字符——字符串结束标志位，判断字符串的长度也依赖于这个 '\0'。

---

## 相关命令

### set



### setex



### setnx



### setrange







### get



### append



### 



---

## 参考

1. https://redis.io/technology/data-structures/
2. https://www.runoob.com/redis/redis-data-types.html
3. https://redis.io/docs/latest/commands/?group=string
