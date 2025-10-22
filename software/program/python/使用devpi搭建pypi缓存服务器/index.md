---
title: "使用devpi搭建pypi缓存服务器"
date: 2025-10-22T15:00:00+08:00
summary: "除了镜像源，我们也许还需要一个缓存服务器"
---

## 目录

[TOC]

---

## 前言

由于 GFW 存在的原因，导致很多仓库无法正常下载，所以一旦涉及到包的下载问题，第一件事情通常就去搜索国内的镜像源,比如：apt、maven、pypi、docker等等。

使用了镜像源之后，我们在开发过程中才不至于捶胸顿足，不过使用Python 的 pypi 的过程中，我发现以下两个问题：

1. pip 没有像maven 一样，有很完善的缓存机制
2. 大厂镜像源也有可能会被请喝茶（比如：docker）

那么在个人服务器搭建一个pypi的缓存服务器就很有必要了。

---

## devpi

在搜索的过程中，发现[devpi](https://devpi.net)比较符合我的需求。

接下来就用该工具在个人云服务器上搭建一个 PyPI 的缓存服务。

### 安装 devpi-server

根据官方文档，可以使用传统的 pip 安装[devpi-server](https://pypi.org/project/devpi-server/)，但是考虑到它作为服务器端的服务，需要在shell里使用相关的命令，我使用 uv tool 安装 devpi-server

```shell
uv tool install devpi-server

koril@ali-djhx-debian:~$ which devpi-server
/home/koril/.local/bin/devpi-server
```
安装好后，执行以下两个命令：

```
# 初始化 devpi-server
devpi-init

# 生成模板配置文件
devpi-gen-config
```

官方使用的是supervisord 管理服务，不过我一般是用systemctl，生成的配置文件里有官方提供的 service 文件，所以直接拷贝到 /etc/systemd/system 下即可：

```
sudo cp gen-config/devpi.service /etc/systemd/system/
```

systemctl 开机自启、启动服务、查看状态：

```
sudo systemctl enable devpi.service
sudo systemctl start devpi.service
sudo systemctl status devpi.service
```

默认状态下，devpi 监听本地的 3141，如果想要使用的时候不加端口号，可以考虑申请个子域名，比如 mirror.yourdomain.com

把 gen-config 下的nginx-devpi.conf 复制到 /etc/nginx/conf.d 下，然后修改其中的server\_name, 改成自己的域名即可。

到此为止，devpi-server 的 systemd 和 nginx 就配置好了，接下来需要使用 devpi-client 来修改上游的源。

### 安装devpi-client

使用 uv tool 安装 devpi-client:

```shell
uv tool install devpi-client
```

如果不修改上游源，默认 devpi-server 是去拉取 pypi.org 的官方源。

devpi-client 登录：

```shell
devpi use http://localhost:3141
devpi login root --password=''
```

查看和使用

```shell
devpi use -l
devpi use root/pypi
```

修改mirror地址为阿里云的源

```shell
devpi index root/pypi mirror_url=https://mirrors.aliyun.com/pypi/simple/
devpi index root/pypi mirror_web_url_fmt=https://mirrors.aliyun.com/pypi/simple/{name}/
```

修改完后，请求页面可以看到修改成功了：

```shell

koril@ali-djhx-debian:~$ curl localhost:3141
{
  "result": {
    "root": {
      "created": "2025-10-22T03:36:19Z",
      "indexes": {
        "pypi": {
          "type": "mirror",
          "volatile": false,
          "title": "PyPI",
          "mirror_url": "https://mirrors.aliyun.com/pypi/simple/",
          "mirror_web_url_fmt": "https://mirrors.aliyun.com/pypi/simple/{name}/"
        }
      },
      "username": "root"
    }
  },
  "type": "list:userconfig"
}

```

配置好后就可以使用了，本地测试一下：

```
python -m pip install -i http://mirror.yourdomian.com/root/pypi/+simple/ --trusted-host mirror.yourdomain.com requests
```

---

## 参考

1. https://zhuanlan.zhihu.com/p/477358157
