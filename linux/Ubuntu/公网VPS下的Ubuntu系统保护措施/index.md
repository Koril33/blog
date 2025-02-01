---
title: "公网VPS下的Ubuntu系统保护措施"
date: 2023-05-04T20:21:41+08:00
tags: []
featured_image: "images/background.jpg"
summary: "公网下的VPS面临着各种脚本的扫描，做好系统的保护非常重要"
toc: true
---

## 前言

公网下的 VPS 每天被各种脚本扫描，为了避免被黑客入侵服务器，做好系统防护非常重要，本文简单的讲述下，从几个不同的方面来保护系统安全。

本文所使用的操作系统是 Ubuntu 20.04.6 LTS。

---

## 登录服务器的要素

当我们在网络上定位某个服务器的某个具体的服务时，需要知道服务器的 IP 地址，端口号，而真正进入服务器，或者操作一些资源和接口，需要额外的用户名和密码。

所以，IP 地址，端口号，用户名和密码这四点就构成了我们保护系统最关键的要素。

IP 地址和端口号，默认都是公开已知的信息，尤其是在一些服务，如 Web 服务，SSH 服务，FTP 服务不更改默认端口号的情况下，这两个要素每分每秒都是暴露在公网上的。

而用户名如果不进行设置，默认拥有 root 用户，那么黑客唯一不知道就是密码了。

---

## 更改默认端口号

因为 IP 是公开的，所以我们只能先从端口下手，例如 SSH，默认端口号是 22，如果我们修改了 SSH 的端口号，那么一部分的针对 22 端口的扫描脚本就失效了。

SSH 的配置文件是 /etc/ssh/sshd_config 

```shell
sudo vim /etc/ssh/sshd_config
```

修改其中的 Port 参数

```
# Port 22
Post 18888
```

Linux 的端口范围在 0-65535 之间，0-1023 用于一些特定的服务，一般不会绑定在这个范围内，除此之外，往大了挑选就行。

然后，保存退出，先别重启 SSH，先额外开一个 SSH 会话，避免待会重启 SSH 之后仅有的会话断开连接，就没办法连上了。

重启 SSH 服务：

```shell
sudo systemctl restart sshd
```

之后通过 SSH 连接服务器，都只能使用刚刚配置的端口了。

---

## 更改当前密码

如果当前的密码不够复杂的话，黑客很容易试出来，所以要更换复杂一些的密码。

更改当前用户的密码：

```shell
passwd
```

---

## 添加一个普通用户

root 作为最高权限的用户，是很多系统自带的默认用户名，这样显然太危险，一旦 root 用户出现安全问题，整个系统就只能待人宰割。

root 用户在系统中拥有绝对权力，执行的任何操作都对系统至关重要，该帐户也可能因意外、恶意或人为无视规则而被不当或不当使用而被滥用，很多时候大家使用 root 来操作仅仅是因为它很“方便”，不需要额外输入密码确认，却可以访问和修改所有的文件。

所以，我们需要新增一个普通用户，用于日常的登陆和不涉及高权限的操作：

```shell
adduser bob
```

然后记得给 bob 也设置一个安全的密码。

---

## 禁止 root SSH 登录

这是另外一个强有的安全措施，可以直接避免黑客尝试使用 root 用户进行 SSH 登录。

```shell
sudo vim /etc/ssh/sshd_config
```

将 PermitRootLogin 设置为 no 即可。

改完后，和刚刚更改 SSH 端口一样，为了保证不会失联，请不要关闭当前的 SSH 登录窗口，而是另外开一个窗口来测试，最后重启即可。

经过以上对 SSH 的配置，之后只能通过以下的命令来登录 SSH了：

```shell
ssh bob@IP地址 -p 18888
```

这样黑客不仅需要猜测端口，还要猜测用户名（因为我们禁止使用 root 直接 SSH 登录），最后再尝试去破解密码，安全系数大大提高。

虽然 root 无法 SSH 登陆了，但平时需要使用 root 权限的时候，使用以下命令即可切换到 root：

```shell
su root
# 输入 root 密码
```

---

## 启动防火墙

Ubuntu 自带了一个叫 ufw 的防火墙，全称为Uncomplicated Firewall，由于 LInux 原始的防火墙工具 iptables 过于繁琐，所以 ubuntu 系统默认提供了一个基于 iptables 之上的防火墙工具 ufw。

### 查看防火墙状态

防火墙一般是默认关闭的。

```shell
sudo ufw status
```

### 配置防火墙

首先，让 ufw 默认禁止所有的传入连接，并允许所有的出站连接，这样相当于外面访问不了服务器，但是服务器能访问外网。

```shell
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

### 允许 SSH 连接

当然，服务器肯定是需要响应外部的一些请求的，这样才能提供服务，尤其是 SSH 连接。

```shell
sudo ufw allow ssh
```

这将创建防火墙规则，以允许端口 22 上的所有连接，这是 SSH 守护进程默认侦听的端口。UFW 知道端口允许 ssh 是什么意思，因为它在 /etc/services 文档中被列为服务。

如果我们为 SSH 连接配置了其他端口（就像之前做的那样），则需要另外指定规则，比如 18888 端口作为新的 SSH 端口：

```shell
sudo ufw allow 18888/tcp comment 'ssh service'
```

### 启动防火墙

如果配置完成了，我们就可以启动防火墙：

```shell
sudo ufw enable
```

### 禁用防火墙

禁用后，创建的任何规则将不再有效。

```shell
sudo ufw disable
```

### 重启防火墙

如果防火墙正在运行，而你又修改了一些规则，可以使用重启命令：

```shell
sudo ufw reload
```

### 删除规则

删除规则可以通过编号删除，使用以下命令可以查看规则的编号：

```shell
sudo ufw status numbered
```

然后就可以删除不需要的规则，比如编号为 2 的规则：

```shell
sudo ufw delete 2
```

---

## 参考

1. https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-20-04
2. https://blog.laoda.de/archives/how-to-secure-a-linux-server#1%E5%AE%89%E8%A3%85ufw%E9%98%B2%E7%81%AB%E5%A2%99
3. https://help.ubuntu.com/community/UFW
