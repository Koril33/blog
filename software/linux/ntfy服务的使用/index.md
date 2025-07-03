---
title: "ntfy服务的使用"
date: 2025-07-03T17:37:00+08:00
summary: "ntfy，一个好用的通知服务"
---

## 目录

[TOC]

---

## 前言

消息推送服务，在运维、爬虫、游戏挂机这些场景经常用到，除了微信公众号的推送（比如：[虾推啥](https://xtuis.cn/)），自建的服务也是很好的选择，比如本文介绍的 ntfy。

官网：https://github.com/binwiederhier/ntfy

## 服务端部署

参考官网部署步骤：

```shell
wget https://github.com/binwiederhier/ntfy/releases/download/v2.12.0/ntfy_2.12.0_linux_amd64.tar.gz
tar zxvf ntfy_2.12.0_linux_amd64.tar.gz
sudo cp -a ntfy_2.12.0_linux_amd64/ntfy /usr/local/bin/ntfy
sudo mkdir /etc/ntfy && sudo cp ntfy_2.12.0_linux_amd64/{client,server}/*.yml /etc/ntfy
sudo ntfy serve
```

如果使用 systemctl 管理，service 文件示例：

```
/etc/systemd/system/ntfy.service

[Unit]
Description=ntfy server
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/ntfy serve --config /etc/ntfy/server.yml
Restart=on-failure
WorkingDirectory=/etc/ntfy
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target

```

server.yml 可以更改端口和域名：



---

## 参考

1. https://github.com/binwiederhier/ntfy
2. https://docs.ntfy.sh/install/#linux-binaries