---
title: "命令行设计的艺术"
date: 2025-11-17T17:01:23
summary: "如果屏幕和键盘一直存在的话，CLI也许永远也不会被淘汰"
---

## 目录

[TOC]

---

## 前言

上世纪七、八十年代诞生的命令行工具在现在的用户看来似乎有些陈旧和繁冗：黑底白字，复杂的帮助手册，莫名其妙的报错，以及需要敲击完整命令所带来的较高门槛，CLI（Command Line Interface）似乎远远没有 GUI（Graphical User Interface）来的那么直观和平易近人。在 GUI 中，大部分的操作都是通过点击、拖拽、框选等鼠标动作来完成，在设计合理的情况下，能大幅降低使用者的学习复杂度。

但这并不意味着命令行工具会被取代，相反它似乎会永远存在，因为相比 GUI，它更适合脚本化，自动化，以及能够以最低的资源占用完成非常复杂和多样的操作。所以 GUI 和 CLI 并非相互取代，而是取决于工具产品的定位，如果一个产品面向大众或者非计算机领域的工程师，那么 GUI 的直观操作无疑是最好的选择，而如果这个产品面向的是运维人员，软件开发者或者计算机爱好者，那 CLI 也许更加合适。

CLI 和 GUI 都是一种交互方式，它们需要封装的是计算机指令的复杂性，当我们为了完成一次图片的处理，比如：图像的二值化，对于底层来说是操作每个像素的值，等于或高于阈值的变成 1，低于阈值的变成 0，但是对于用户而言，他们不可能为了处理图片而专门学习计算机的编程语言，所以需要对业务逻辑进行封装。

GUI 会选择展示图片，并且提供菜单选项和按钮，让用户就像是去饭店点餐一样直观简单。

CLI 则通过终端，用户需要键入指令，传入的图片路径，操作命令，以及输出的图片保存路径。

GUI 会占用更多系统资源，并且难以自动化，好处就是简单直观，而 CLI 的资源占用往往很小，并且适合自动化，但缺点就是有较高的学习门槛。

---

## 参考

1. https://sunbk201.github.io/cli-guidelines-zh/
2. https://clig.dev/
3. https://medium.com/@jdxcode/12-factor-cli-apps-dd3c227a0e46
4. https://typer.tiangolo.com/
5. https://zhuanlan.zhihu.com/p/593598678
6. https://news.ycombinator.com/item?id=25304257
7. https://blog.joway.io/posts/the-art-of-cli-design/
8. https://blog.atlan.com/engineering/the-art-of-building-delightful-clis-lessons-learned-from-building-the-atlan-cli/