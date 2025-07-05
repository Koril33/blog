---
title: "TextEditor中文输入重复显示的BUG"
date: 2025-07-05T16:31:00+08:00
summary: "TextEditor中文输入重复显示的BUG"
---

## 问题

Text Editor 编辑文本文件会出现中文字符输入完成后或者创建文件夹，中文重复显示：

![](./images/1.png)

![](./images/2.png)

![](./images/3.png)

---

## 解决

输入法设置里面，改成兼容模式：

![](./images/4.png)

![](./images/5.png)

---

## 参考

1. https://github.com/libpinyin/ibus-libpinyin/issues/469
