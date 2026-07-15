---
title: "Shell的快捷键"
date: 2026-05-24T09:52:08
summary: "提高 Bash Shell 使用体验的快捷键"
---

## 目录

[TOC]

---

## 移动

除了方向键这种，最简单的左右单字符移动外，在碰到单行长命令时，需要更高效的移动手段。

跳到行首：Ctrl + A
跳到行尾：Ctrl + E

跳到之前一个（向左）的单词：Alt + B
跳到之后一个（向右）的单词：Alt + F

---

## 删除

除了 Backspace 这种删除单个字符之外，还可以快速删除多字符。

删除光标左边一个字符：Ctrl + H（等同于 Backspace）
删除光标右边一个字符：Ctrl + D

删除光标左边一个单词：Ctrl + W
删除光标右边一个单词：Alt + D

删除光标至行首：Ctrl + U
删除光标至行尾：Ctrl + K

---

## 剪切和粘贴

Bash 里没有 Windows 那种 Ctrl+C / Ctrl+V 逻辑。内部叫 Kill（剪切） 和 Yank （粘贴）。

刚刚提到的删除都是剪切，可以通过 Ctrl + Y 来粘贴上一次剪切的内容。

终端里的复制粘贴命令是：

复制：Ctrl + Shift + C
粘贴：Ctrl + Shift + V

---

## 历史命令移动

上一条命令：Ctrl + P（等同于方向键的↑）
下一条命令：Ctrl + N（等同于方向键的↓）

---

## 搜索历史

Bash 支持 reverse-i-search（反向增量搜索），因为使用方向键的上下找不够快，所以可以使用 Ctrl + R 调用反向增量搜索。

优先从最近的一条命令匹配，多次按下 Ctrl + R 可以往历史的深处继续寻找，找到后可以使用 Ctrl + E 退出进行修改，也可以不退出，直接回车执行。

Ctrl + E 是退出并修改，如果不想要修改使用 Ctrl + G 或者 Ctrl + C。

如果想要更加现代化的体验，可以安装 fzf 工具：

```shell
sudo apt install -y fzf
```
安装好后，把以下两行加入 `~/.bashrc`：

```shell
source /usr/share/doc/fzf/examples/key-bindings.bash
source /usr/share/doc/fzf/examples/completion.bash
```
然后：

```shell
source ~/.bashrc
```
