---
title: "frp服务的使用"
date: 2025-07-19T14:03:00+08:00
summary: "frp，一个好用的内网穿透工具"
---

## 目录

[TOC]

---

## 前言

在内网环境下，主机都是与互联网隔绝的，如果我们想要在公网访问一个内网的设备，其中一个办法就是使用内网穿透技术，它解决了家庭宽带、企业局域网等环境下无法从外部直接访问内部服务的问题。

**FRP（Fast Reverse Proxy）** 是一个高性能的开源内网穿透工具，使用 Go 语言开发。它通过在公网服务器上运行 `frps`（服务端），在内网设备上运行 `frpc`（客户端），建立反向连接，实现从公网访问内网服务（如 SSH、Web、数据库等）。配置灵活，支持 TCP、UDP、HTTP、HTTPS 等多种协议，适用于远程管理、自建服务、IoT 等场景。

---

## 下载

在 Github release 页面下载最新的版本：

https://github.com/fatedier/frp/releases

根据服务器系统，选择对应的压缩包，比如我下载的是：[frp_0.63.0_linux_amd64.tar.gz](https://github.com/fatedier/frp/releases/download/v0.63.0/frp_0.63.0_linux_amd64.tar.gz)

解压后，有以下文件：

```
frpc  frpc.toml  frps  frps.toml  LICENSE
```

frpc 和 frpc.toml 是客户端的程序和配置文件模板。

frps 和 frps.toml 是服务器端的程序和配置文件模板。

---

## 服务端部署

服务器需要至少打开一个端口（防火墙或者安全组）的外网访问。

服务器和客户端就是通过这个端口（官方默认是 7000）建立连接。

把 frps 和 frps.toml 放置到服务器的目录下（比如：/opt/frp），然后修改配置文件 frps.toml：

```toml
# 必须要配置的
bindPort = 7000
auth.token = "your password"

# dashboard 可有可无
webServer.addr = "127.0.0.1"
webServer.port = 7500

webServer.user = "admin"
webServer.password = "your password"
```

bindPort 就是服务器与客户端通信的端口，auth.token 表示连接的验证 token。

如果没有这个 token，任何人只要知道你的 frps 的公网 IP 和端口，就可以连接你的 frp 服务，所以一定要指定一个 auth.token。

webServer 的配置，主要是给服务器端一个可以查看的 web dashboard 页面，可有可无。

配置好文件后，这里可以把 frps 加入到 systemd 进行管理，官网的 systemd service 有些简单，没有指定用户和重启策略，我的版本是：

```ini
# /etc/systemd/system/frps.service
[Unit]
Description=Frp server
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=60s
StartLimitBurst=3

[Service]
Type=simple
WorkingDirectory=/opt/frp
User=frp
Group=frp
ExecStart=/opt/frp/frps -c /opt/frp/frps.toml
ExecStop=/bin/kill -s TERM $MAINPID
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

这里我指定运行的用户和属组是 frp，所以还需要建立一个专门运行 frp 服务的系统级用户（和 nginx 的 www-data 类似）：

```sh
sudo useradd --system --no-create-home --shell /usr/sbin/nologin frp
```

然后把 /opt/frp 的目录权限指定给 frp 用户：

```sh
sudo chown -R frp:frp /opt/frp
```

到此，所有服务器端的 frps 准备工作完成，启动 frps 服务：

```sh
sudo systemctl daemon-reload
sudo systemctl enable frps.service
sudo systemctl start frps.service
```


---

## 客户端部署

客户端和服务器端的部署几乎一模一样，把 frpc 和 frpc.toml 放到 /opt/frp 下，然后配置 frpc.service：

```ini
# /etc/systemd/system/frpc.service
[Unit]
Description=Frp client
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=60s
StartLimitBurst=3

[Service]
Type=simple
WorkingDirectory=/opt/frp
User=frp
Group=frp
ExecStart=/opt/frp/frpc -c /opt/frp/frpc.toml
ExecStop=/bin/kill -s TERM $MAINPID
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

然后创建 frp 用户，更改 /opt/frp 权限给 frp 用户，唯一不同的是 frpc.toml 的内容，如下：

```toml
serverAddr = "your frps server ip address"
serverPort = 7000
auth.token="your password"

[[proxies]]
name = "my-ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 7001
```

serverAddr 就是部署了 frps 服务器的 ip 地址或者域名，serverPort 和 auth.token 和服务器 frps.toml 里面保持一致，下面的 [[proxies]] 小节就是专门配置代理本地的服务到公网去。

这里的 proxies 配置是把内网机器的 ssh 的 22 端口映射到公网服务器的 7001 端口。

配置好 frpc.toml 后，启动：

```sh
sudo systemctl daemon-reload
sudo systemctl enable frpc.service
sudo systemctl start frpc.service
```

验证 ssh 是否可以连接，假设我这里的公网 ip 是 1.2.3.4，ssh 用户名是 alice，那么 ssh 命令如下：

```sh
ssh -p 7001 alice@1.2.3.4
```

---

## 参考

1. https://github.com/fatedier/frp
2. https://gofrp.org/zh-cn/docs/features/common/ui/
3. https://gofrp.org/zh-cn/docs/setup/systemd/
