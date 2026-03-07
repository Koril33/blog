---
title: "SQLAlchemy入门"
date: 2026-02-06T10:21:53
summary: "Python 生态里的 ORM 框架"
---

## 目录

[TOC]

---

## SQLAlchemy 基本概念

SQLAlchemy 给 Python 生态提供了一个完整的 SQL 和 ORM 工具包。SQLAlchemy 最重要的两个前端组件是对象关系映射器 (ORM) 和核心组件（Core）。

Core 主要提供了数据库连接的抽象，数据库连接池的管理，以及最重要的 SQL 构造器，这是一种以 Python 语言的方式来书写 SQL 的模型，可以让用户不必手写原生的 SQL 代码。

ORM 则是用 Python 的面向对象和数据库进行映射的框架。这是一种针对领域对象建模的方式，通常用于后台系统建模。用户能以对象为核心进行增删改查，ORM 是建立在 Corn 之上的更高层面的抽象和封装。

---

## 



---

## 参考

1. https://docs.sqlalchemy.org/en/20/intro.html
2. https://docs.sqlalchemy.org/en/20/tutorial/index.html
