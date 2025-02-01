---
title: "VirtualBox中虚拟机运行缓慢的解决方案"
date: 2024-06-13T18:10:59+08:00
tags: []
featured_image: "images/background.jpg"
summary: "虚拟机缓慢的解决方案"
toc: true
---

## 前言

关于 VirtualBox 运行缓慢的原因，网上说法很多，这里简单总结下。

---

## 原因一

可能是资源给少了，比如 CPU 和内存。

参考：[virtualbox优化配置（解决卡顿）_virtualbox 卡顿-CSDN博客](https://blog.csdn.net/csdncaozhenming/article/details/118642772)

---

## 原因二

我的一台电脑是 2024 年刚买的，配置都很高，而且分配给虚拟机的资源也不少，电脑到手后一开始运行 VirtualBox 虚拟机的时候很流畅，但是从某一天开始后，就变得很卡，始终没找到原因，网上的资料也都是抄来抄去的。

直到阅读到了这篇博客：[VirtualBox中Windows7虚拟机运行缓慢的解决办法 | Tsccai's Blog](https://tsccai.github.io/2022/06/11/solution-to-slow-win7-on-virtual-box/)

才发现可能是因为那一天启动了 Windows 的 WSL2，所以开启了 Hyper-V，而产生冲突。

所以解决方案就是关闭 Windows 的 Hyper-V。

关闭的方法参考：[Win11无法关闭hyper-V的解决方案（详细教学） - 哔哩哔哩 (bilibili.com)](https://www.bilibili.com/read/cv17864186/)
