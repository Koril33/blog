---
title: "ntfy服务的使用"
date: 2025-07-03T17:37:00+08:00
summary: "ntfy，一个好用的通知服务工具"
---

## 目录

[TOC]

---

## 前言

消息推送服务，在运维、爬虫、游戏挂机这些场景经常用到，除了微信公众号的推送（[虾推啥](https://xtuis.cn/)），自建的服务也是很好的选择，比如本文介绍的 ntfy。

官网：https://github.com/binwiederhier/ntfy

---

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

server.yml 可以更改端口和域名，假设你的域名是 notify.test.com，本地端口是 18080

```yaml
# ntfy server config file
base-url: "notify.test.com"
listen-http: "127.0.0.1:18080"
upstream-base-url: "https://ntfy.sh"
```


为了开启 websocket，Nginx 配置如下：

```

server {
    listen 80;
    server_name notify.test.com;

    location / {
        proxy_pass http://127.0.0.1:18080;

        # WebSocket 所需的头部
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # 常规头部
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

```

更新 nginx

```
sudo nginx -t
sudo nginx -s reload
```

启动 ntfy 服务

```
sudo systemctl daemon-reload
sudo systemctl start ntfy.service
sudo systemctl enable ntfy.service
```

---

## 访问控制

ntfy 默认情况下，允许所有人发送和接收任何 topic 信息，所以任何人都可以使用你自建的 ntfy 服务器发送任何消息。

ntfy 支持用户权限控制，使用的模型是 ACL（Access Control List）。

配置文件添加以下两行配置：

```
auth-file: "/var/lib/ntfy/user.db"
auth-default-access: "deny-all"
```

这里配置了 auth-default-access 表示默认没有在数据库中找到匹配的用户，一律禁止访问。

重启服务后，会自动创建 user.db 文件（sqlite3 数据库），先添加一个 admin 角色的用户：

```shell
sudo ntfy user add --role=admin bob
```

终端会提示输入密码，输入完后，就成功创建了一个 admin 权限的用户，该用户可以读取和写入所有 topic。

也可以创建一个普通用户（角色是 user），命令如下：

```shell
sudo ntfy user add alice
```

创建完普通用户后，由于没有分配权限，所以无法访问任何的 topic，通过以下命令可以设置权限，比如允许 alice 读写名为 test 的 topic：

```shell
sudo ntfy access alice test rw
```

更加详细的 ACL 配置，参考官网文档：https://docs.ntfy.sh/config/#access-control

---

## 参考

1. https://github.com/binwiederhier/ntfy
2. https://docs.ntfy.sh/install/#linux-binaries
3. https://k1r.in/posts/notify-ntfy/
4. https://shuaixin.cc/ntfy/