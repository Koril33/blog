---
title: "Docker挂代理"
date: 2025-02-06T21:03:00+08:00
summary: "Docker如何挂代理"
---

## 问题

Docker 的镜像源，由于众所周知的原因不能使用了，所以下载镜像需要挂上代理。

---

## 解决

在`/etc/docker`下新建文件——daemon.json，写入以下内容：

```json
{
  "proxies": {
    "http-proxy": "http://127.0.0.1:7890",
    "https-proxy": "http://127.0.0.1:7890",
    "no-proxy": "*.test.example.com,.example.org,127.0.0.0/8"
  }
}
```

然后重启 docker:

```
sudo systemctl restart docker
```
