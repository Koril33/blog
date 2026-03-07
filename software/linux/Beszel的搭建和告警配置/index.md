---
title: "Beszel的搭建和告警配置"
date: 2026-02-27T14:02:51
summary: "一个更加轻量、简洁的监控系统"
---

## Beszel

Beszel 是一个轻量级的服务器监控平台，包含 Docker 统计信息、历史数据和警报功能，使用 Go 开发，支持跨平台。

Beszel 主要有两个组件：

- 中心（Hub）：仪表盘，查看和管理系统
- 代理（Agent）：代理在要监控的每个系统上运行，并将系统指标传递给中心

---

## 安装

Beszel 支持 Docker 或者二进制安装，参考：

Hub：https://www.beszel.dev/zh/guide/hub-installation

Agent：https://www.beszel.dev/zh/guide/agent-installation

首先安装并启动 Hub，然后在 http://your_ip:8090 的 web 页面上创建第一个管理员账号：

![](./images/1.jpg)

进入系统后，右上角点击“添加客户端”：

![](./images/2.jpg)

把安装命令复制到需要监控的客户端服务器上运行，客户端执行完命令后，回到 Hub Web 页面，点击添加客户端。

---

## 告警

这里以邮件告警为示例，其他的告警方式可以参考官方文档。

![](./images/3.jpg)

### 配置 SMTP

把自己开通的 SMTP 服务器的地址，端口，用户名和密钥填写在 Mail setttings 中：

![](./images/4.jpg)

配置好后，回到首页，对需要进行告警配置的服务器，点击小铃铛：

![](./images/5.jpg)

可以对不同的指标进行告警配置：

![](./images/6.jpg)

---

## 参考

1. https://www.beszel.dev/zh/guide/getting-started
2. https://juejin.cn/post/7582479325284548671
3. https://www.cnblogs.com/minseo/p/18728811
4. https://blog.csdn.net/gitblog_00075/article/details/151310248