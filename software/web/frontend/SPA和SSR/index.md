---
title: "SPA和SSR"
date: 2026-02-01T10:48:21
summary: "不同的前端方案"
---

## 前言

自前后端分离流行以来，前端的主流方案有两种，一种叫 SPA，另一种叫 SSR，本文简述二者的概念与区别。

---

## SPA

SPA，Single Page Application，即单页面应用。根据渲染的位置来看，也可以称其为客户端渲染（CSR，Client Side Render）。

和传统的前后端不分离的模板引擎（比如 Java 的 Thymeleaf 或者 Python 的 Jinja）不同，SPA 的原理是：浏览器初次请求只加载一个 HTML 壳子和 JS 脚本。后续所有的页面跳转和数据更新都通过 Ajax/Axios 请求后端的 JSON API，由前端动态渲染 DOM。

也就是说，所有的数据更新导致的页面控件变化，全部由客户端浏览器的 JavaScript 请求并更新。

缺点是

