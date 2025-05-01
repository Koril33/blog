---
title: "Linux的grep命令展示匹配项的前后n行信息"
date: 2023-07-03T17:27:59+08:00
tags: []
featured_image: "images/background.jpg"
summary: "grep 展示匹配项的前后 n 行的信息"
toc: true
---

## 前言

在工作中经常使用 grep 来查看日志文件的匹配信息，但是不加任何参数的话，仅能打印匹配行，但有的时候，希望能够展示匹配行的前 n 行，后 n 行信息。

---

## grep -A

-A 参数（--after-context），除了显示符合范本样式的那一行之外，并显示该行之后的内容。

---

## grep -B

-B 参数（--before-context），除了显示符合样式的那一行之外，并显示该行之前的内容。

---

## grep -C

-C 参数（--context），除了显示符合范本样式的那一列之外，并显示该列之前后的内容。
