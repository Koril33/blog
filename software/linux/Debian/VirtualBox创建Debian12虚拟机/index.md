---
title: "VirtualBox创建Debian12虚拟机"
date: 2024-11-16T09:07:50+08:00
tags: []
featured_image: "images/background.jpg"
summary: "VirtualBox 安装 Debian12 以及设置静态 IP 地址，安装一些简单的软件"
toc: true
---

## 前言

如果让我选的话，我会在个人项目和公司项目都选用 Debian，网上的讨论也很多：

[公司 CentOS 服务器需更换系统，选择 Debian or Ubuntu？ - V2EX](https://www.v2ex.com/t/1054147)

[Debian 还是 Ubuntu server - V2EX](https://v2ex.com/t/978670)

Debian 的稳定和轻量，很适合不折腾的用户。

本文从下载镜像开始，逐步讲解如何在本地宿主机上安装一个 Debian 的虚拟机系统。

---

## 环境

网络：家庭网络

宿主机：ThinkBook 16+

宿主机系统：Windows 11

虚拟机软件：VirtualBox 7

需要安装的虚拟机：Debian 12（截至目前，最新版本是 12.8.0）

---

## 镜像

Debian 官方的下载目录页：[下载 Debian](https://www.debian.org/distrib/)

### 选择

Debian 根据不同的环境和需求准备了大量的安装镜像：

- Live ISO：Live 镜像可以直接从 USB 或 DVD 启动并进入 Debian 系统，无需安装，适合试用或故障排查。安装过程通常提供图形化界面（例如 GNOME、KDE 等）。
- USB 镜像：为 USB 设备专门优化的 ISO，适合直接烧录到 USB 驱动器进行安装或启动。
- CD/DVD 镜像：Debian 的完整安装镜像，包含大量软件包，适合在没有网络或网络较慢的情况下离线安装。DVD 镜像通常包含更多软件包。
- netinst（网络安装）：只有一个小型的基础 ISO 镜像（通常仅 300 MB 左右），安装时需要联网下载所需的软件包，适合网络稳定的环境。

本文用的是 CD/DVD 格式的 ISO 文件。

选择好 ISO 类型后，下一步就是去下载 ISO 文件，可以选择官方的下载源，也可以选择对应国家的下载镜像源。

官方提供的地址：[/debian-cd/current/amd64/iso-dvd 的索引](https://cdimage.debian.org/debian-cd/current/amd64/iso-dvd/)

![](./images/1.jpg)

也可以选择对应国家的镜像源进行下载：[通过 HTTP/FTP 下载 Debian USB/CD/DVD 映像](https://www.debian.org/CD/http-ftp/#mirrors)

![](./images/2.jpg)

![](./images/3.jpg)

以兰州大学镜像源为例：

- 12.8.0/：表示 Debian 12 的正式版（具体版本号为 12.8.0），包含标准的安装 ISO 镜像。
- 12.8.0-live/：Debian 12.8.0 的 Live 版本，适合试用或不需安装的场景。包含可直接运行的桌面环境（如 GNOME、KDE、XFCE 等）。
- current/：这是当前最新的 Debian 稳定版本的快捷方式。访问 current/ 就相当于访问 12.8.0。
- current-live/：当前最新的 Live 版本快捷方式，与 12.8.0-live/ 指向相同的内容。
- project/：该目录通常用于存放关于 Debian 项目本身的文件和信息，如开发日志、历史版本信息等。
- ls-lR.gz：是一个压缩的文本文件，包含整个目录树的文件列表，方便快速查找目录下所有文件。

选择 12.8.0/ 或者 current/ 目录，选择对应架构的目录：

![](./images/4.jpg)

可以看到有很多不同的形式的下载目录：

![](./images/5.jpg)

格式说明如下：

1. **bt-bd / bt-cd / bt-dvd**

- “bt” 表示 BitTorrent，文件夹中的内容是通过 BitTorrent 下载的镜像种子文件（torrent）。
- `bd` 表示蓝光光盘（Blu-ray Disc）。
- `cd` 和 `dvd` 分别代表 CD 和 DVD 安装盘镜像。
- 这些文件夹提供了不同容量和格式的镜像下载，以满足使用 BitTorrent 下载的用户需求。

2. **iso-bd / iso-cd / iso-dvd**

- “iso” 表示标准的 ISO 镜像文件。
- `bd` 目录下是蓝光光盘的 ISO 镜像。
- `cd` 目录下是 CD 格式的 ISO 镜像，适合较小容量的安装需求。
- `dvd` 目录下是 DVD 格式的 ISO 镜像，包含更多的软件包，用于完整安装。

3. **jigdo-16G / jigdo-bd / jigdo-cd / jigdo-dlbd / jigdo-dvd**

- **Jigdo（Jigsaw Download）** 是一种下载工具，适合分段下载 ISO 镜像，减少带宽使用。适用于断点续传或重新下载镜像更新。
- `16G` 和 `bd` 适用于蓝光光盘或容量大的镜像。
- `cd` 和 `dvd` 提供较小的安装镜像，方便按需下载。
- `dlbd` 表示 “双层蓝光光盘”，支持更大容量的蓝光镜像。

4. **list-16G / list-bd / list-cd / list-dlbd / list-dvd**

- 这些目录中的文件是包含不同镜像内容的文件列表（例如软件包和文件的索引），方便用户了解每个 ISO 镜像所包含的内容。
- `16G` 和 `bd` 文件列表适合大容量的安装镜像。
- `cd` 和 `dvd` 提供的是适用于普通 CD 和 DVD 镜像的内容清单。

5. **log**

- `log` 目录下通常包含生成或更新这些镜像的日志文件，记录了镜像的构建和更新历史。

这里选择 iso-dvd 版本即可：

![](./images/6.jpg)

### 校验

下载成功后，可以进行文件校验：[验证 Debian 映像的真实性](https://www.debian.org/CD/verify)

这一步可以也略过，直接进入安装步骤。

我用的是 Windows 11，所以可选择 Powershell 或者 Git-Bash（安装过 git），来进行校验。

本文使用 Git-Bash 中的 `gpg` 和 `sha512sum` 来进行文件校验。

`SHA512SUMS` 和 `SHA256SUMS` 是 Debian 提供的两种不同的哈希校验文件，区别在于它们使用的哈希算法不同：

- **SHA512SUMS**：包含文件的 SHA-512 哈希值列表，使用的是 SHA-512 哈希算法。SHA-512 更长且更安全，对完整性验证更强大，但计算速度较慢，占用存储空间较多。
- **SHA256SUMS**：包含文件的 SHA-256 哈希值列表，使用的是 SHA-256 哈希算法。SHA-256 哈希值相对较短，计算速度更快，广泛用于文件完整性验证。

两者都可以用于验证文件完整性和防篡改。根据需求可以选择任意一种，比如如果你需要更高的安全性和完整性验证，可以选择 `SHA512SUMS`，否则 `SHA256SUMS` 已足够。

以 SHA512 为例，下载下面两个文件：

![](./images/7.jpg)

下载结果如下：

![](./images/8.jpg)

```sh
$ gpg --verify SHA512SUMS.sign SHA512SUMS
gpg: Signature made 2024年11月10日  0:35:04
gpg:                using RSA key DF9B9C49EAA9298432589D76DA87E80D6294BE9B
gpg: Can't check signature: No public key
```

显示没有导出 public key，那就去官网获取 public key：

```sh
$ gpg --keyserver keyring.debian.org --recv-keys DF9B9C49EAA9298432589D76DA87E80D6294BE9B
gpg: key DA87E80D6294BE9B: public key "Debian CD signing key <debian-cd@lists.debian.org>" imported
gpg: Total number processed: 1
gpg:               imported: 1
```

接下来这一步是用 SHA512SUMS.sign 这个签名，来验证 SHA512SUMS 是 Debian 官网公布的。

```sh
$ gpg --verify SHA512SUMS.sign SHA512SUMS
gpg: Signature made 2024年11月10日  0:35:04
gpg:                using RSA key DF9B9C49EAA9298432589D76DA87E80D6294BE9B
gpg: Good signature from "Debian CD signing key <debian-cd@lists.debian.org>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: DF9B 9C49 EAA9 2984 3258  9D76 DA87 E80D 6294 BE9B
```

最后再用 SHA512SUMS 来判断 ISO 的完整性：

```sh
$ sha512sum --check --ignore-missing SHA512SUMS
debian-12.8.0-amd64-DVD-1.iso: OK
```

该命令会自动检测到当前目录下的 SHA512SUMS 文件，然后根据文件中的列表来校验 ISO 的完整性。

---

## 虚拟机安装

选择好镜像文件和虚拟机保存地址：

![](./images/9.jpg)

选择内存和核心数：

![](./images/10.jpg)

选择磁盘：

![](./images/11.jpg)

最后，点击完成：

![](./images/12.jpg)

启动刚刚创建的虚拟机：

![](./images/13.jpg)

选择使用图形界面来完成安装：

![](./images/14.jpg)

选择中国-中文

![](./images/15.jpg)

![](./images/16.jpg)

![](./images/17.jpg)

主机名，后面可以再改

![](./images/18.jpg)

![](./images/19.jpg)

root 的密码，因为这里是测试环境，所以随便设置简单的密码。

在正式环境中，一定要使用强密码。

![](./images/20.jpg)

用户名

![](./images/21.jpg)

![](./images/22.jpg)

![](./images/23.jpg)

![](./images/24.jpg)

![](./images/25.jpg)

![](./images/26.jpg)

![](./images/27.jpg)

![](./images/28.jpg)

![](./images/29.jpg)

可以使用国内的镜像站点，加速安装

![](./images/30.jpg)

![](./images/31.jpg)

![](./images/32.jpg)

![](./images/33.jpg)

![](./images/34.jpg)

这里作为服务器，不需要安装图形界面，为了 SSH 连接，提前装好 SSH Server

![](./images/35.jpg)

![](./images/36.jpg)

![](./images/37.jpg)

![](./images/38.jpg)

重启后，能用 root 或者普通用户登陆，就算成功安装好了 Debian12 系统。

![](./images/39.jpg)

![](./images/40.jpg)

---

## 切换用户

Debian 12 默认是没有 sudo 命令，需要自己安装，所以在一开始的时候，如果用普通用户登陆，我们想要执行一些命令没法使用 sudo。

这时候，可以先用命令 `su -` 来切换成 root 用户，加上 - 表示加载完整的用户的 shell 环境。

或者使用 apt 下载 sudo，再把普通用户添加到 sudo 中（后面会介绍）。

---

## APT

首先，确认下可以连接互联网，然后通过 apt 来下载一些基本的工具，方便后续的步骤。

### 源

关于 apt 的 sources.list 的文档，可以参考：[SourcesList - Debian Wiki](https://wiki.debian.org/SourcesList)

apt 的源，在 `/etc/apt/sources.list` 当中。

`/etc/apt` 有两个文件，sources.list 和 sources.list.d，二者区别如下：

- sources.list：用于定义主要的系统软件源。
- sources.list.d/：用于添加和管理第三方源或单独的源文件。

例如，安装 Docker 时，安装脚本可能会在 `/etc/apt/sources.list.d/` 下创建一个名为 `docker.list` 的文件，文件中仅包含 Docker 的软件源地址。

目前，我们只用到 sources.list 就行，由于之前安装的时候，我选择了镜像源，所以我的 sources.list 如下：
```
deb cdrom:[Debian GNU/Linux 12.8.0 _Bookworm_ - Official amd64 DVD Binary-1 with firmware 20241109-11:05]/ bookworm contrib main non-free-firmware

deb http://ftp.cn.debian.org/debian/ bookworm main non-free-firmware
deb-src http://ftp.cn.debian.org/debian/ bookworm main non-free-firmware

deb http://security.debian.org/debian-security bookworm-security main non-free-firmware
deb-src http://security.debian.org/debian-security bookworm-security main non-free-firmware

# bookworm-updates, to get updates before a point release is made;
# see https://www.debian.org/doc/manuals/debian-reference/ch02.en.html#_updates_and_backports
deb http://ftp.cn.debian.org/debian/ bookworm-updates main non-free-firmware
deb-src http://ftp.cn.debian.org/debian/ bookworm-updates main non-free-firmware
```

第一行，表示现在从 cd 光盘中读取软件源，从网络读取的源是最新的，并且第一行只有装上光盘似乎才能使用。

所以需要注释掉第一行，然后使用国内网络源来下载。

关于 apt 镜像源的选择，可以参考：[Debian -- Debian 全球镜像站](https://www.debian.org/mirror/list)

在介绍中，官网提到了一个软件—— [`netselect-apt`](https://packages.debian.org/stable/net/netselect-apt) ，可以用来决定哪个站点有最小的延迟。

安装 netselect-apt，文档：[netselect-apt（1） — netselect-apt — Debian 测试 — Debian 手册页](https://manpages.debian.org/testing/netselect-apt/netselect-apt.1.en.html)

```sh
apt install netselect-apt
```

然后执行命令：`netselect-apt -sn` 

执行后，会在当前执行命令的目录下生成一个 sources.list，然后用这个生成的文件覆盖原始的 /etc/apt/sources.list 即可。

![](./images/44.jpg)

更新 apt 软件源：

```sh
apt update
apt upgrade
```

### 基本软件

```sh
apt install vim curl sudo
```

---

## 添加用户到 sudo

apt 安装好 sudo 后，把刚刚的普通用户添加到 sudo 中。

第一步：切换到 root 用户：

```sh
su - root
# 不加 - 的话，没有加载 root 的环境变量，后面的 usermod 会提示找不到命令
```

第二步：把用户加到 sudo 中：

```sh
usermod -aG sudo koril
```

第三步，重新登陆普通账号，就能用 sudo + 命令了。

---

## 网络

目前虚拟机默认的是 NAT 模式，就是处于宿主机内网中，虚拟机能连接外部网络，但是局域网的其他机器无法连接该虚拟机。

在虚拟机网络中选择 NAT（网络地址转换）模式有以下优缺点：

### 优点

1. **简单设置**：NAT 模式对宿主机的网络环境要求较低，不需要特殊网络配置，虚拟机可以直接访问互联网。
2. **隔离性好**：虚拟机通过 NAT 访问外部网络，但外部网络无法主动访问虚拟机，增强了虚拟机的安全性。
3. **节省 IP 地址**：NAT 模式下，虚拟机与宿主机共享一个 IP 地址，适合公网 IP 数量有限的场景。

### 缺点

1. **外部无法访问虚拟机**：NAT 模式通常用于虚拟机主动访问外部资源，外部无法直接访问虚拟机，影响服务器类应用的部署。
2. **网络性能稍差**：由于需要经过 NAT 转换，NAT 模式会有一些性能损失，尤其在需要高性能网络连接的场景中可能会有影响。
3. **端口映射配置**：如果需要外部访问虚拟机特定端口，必须手动配置端口转发，增加了复杂性。

### 适用场景

NAT 模式适合需要隔离的环境，例如仅需虚拟机访问外部网络、不需要外部访问虚拟机的情况。

作为测试用的服务器，本文选择“桥接网卡”模式，先正常关闭虚拟机，然后在设置中配置网络：

![](./images/41.jpg)

重新启动虚拟机，查看 ip 地址：

![](./images/42.jpg)

这里通过 DHCP 服务器（家里的路由器）自动获取到了一个 IP 地址，由于之前安装好了 SSH Server，已经可以在宿主机或者别的局域网的机器，通过 SSH 连接到该虚拟机了：

![](./images/43.jpg)

但是作为测试服务器，重启后，IP 通过 DHCP 可能会发生改变，服务器最好使用静态 IP 地址，而不是依赖 DHCP。这样可以确保服务器的可访问性和网络配置的稳定性。

### 配置静态 IP

设置静态 IP 要注意，选择的这个 IP 没有被其他局域网设备使用过，可以通过 ping、ip neigh，nmap 等命令确认。 

网络的配置文件在 `/etc/network/interfaces`，内容如下：

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug enp0s3
iface enp0s3 inet dhcp
```

这是第一个很让人困惑的点，究竟用 /etc/network/interfaces 还是 systemd-networkd ？

可参考的观点如下：

1. [/etc/network/interfaces（将要）被弃用了吗？： r/Debian](https://www.reddit.com/r/debian/comments/18nbs9l/is_etcnetworkinterfaces_going_to_be_deprecated/)
2. [linux - 网络配置：systemd-networkd 与网络、服务和与 ip 命令的兼容性 - 超级用户](https://superuser.com/questions/1450668/network-configuration-systemd-networkd-vs-networking-service-and-compatiblity)
3. [网络 - /etc/network/interfaces、systemd-networkd 和 NetworkManager：它们如何共存？- 询问 Ubuntu](https://askubuntu.com/questions/1161422/etc-network-interfaces-systemd-networkd-and-networkmanager-how-do-they-coexis)
4. [Linux 下的网络管理服务太乱了，我现在都没搞明白之间的关系 - V2EX](https://v2ex.com/t/834317)

我也在 v2ex 上提问了，可以参考下大家的回答：https://v2ex.com/t/1110984

systemd-networkd 一开始是没有启动的，但在官方文档（[第 5 章 网络设置](https://www.debian.org/doc/manuals/debian-reference/ch05.zh-cn.html)）的这一节——5.3. 没有图像界面的现代网络配置，当中提到了，现代网络推荐使用 systemd-networkd 配置，故本文决定采用 systemd-networkd。

```sh
# 查看当前到底是用哪个（默认是 networking.service，使用的配置文件是 /etc/network/interfaces）
sudo systemctl status networking.service
sudo systemctl status systemd-networkd.service
sudo systemctl status NetworkManager.service
```

systemd-networkd 的参考文档：[SystemdNetworkd - Debian Wiki](https://wiki.debian.org/SystemdNetworkd)

为了使用 systemd-networkd，我们把 /etc/network 下的 interfaces 改成别的名字，比如：interfaces.bak（不然的话，重启后可能还会走 /etc/network/interfaces 的配置，造成混乱），然后配置`/etc/systemd/network/static.network`：

```
[Match]
Name=en*

[Network]
DHCP=no
Address=192.168.0.202/24
Gateway=192.168.0.1
DNS=192.168.0.1
```

关闭其他的网络服务，然后启动 systemd-networkd：

```sh
# 关闭默认的 networking.service
sudo systemctl disable networking.service
sudo systemctl stop networking.service

sudo systemctl enable systemd-networkd
sudo systemctl start systemd-networkd
```

静态的 IP 地址就配置完毕。

可以通过 iproute2 命令来查看 ip 和路由信息（iproute2 取代 net-tools）：

```sh
ip addr show
ip route show
```

---

## 主机名

### 查看

```sh
hostname
# 或者
hostnamectl
```

### 修改

方法一，hostnamectl 命令：

```sh
sudo hostnamectl set-hostname your-host-name
```

方法二，手动修改文件：

1. /etc/hostname 文件
2. /etc/hosts 文件

---

## 时间

时间，时区的设置，暂略

---

## 语言

暂略

---

## 总结

安装部分，最困难的可能就是：

1. 网络连接问题（无法连接互联网，网络卡顿）
2. APT 镜像源设置错误
3. 配置静态 IP 的问题 


---

## 参考

1. [技术|如何找到合适的 Debian ISO 来下载](https://linux.cn/article-15662-1.html)
1. [Debian 参考手册](https://www.debian.org/doc/manuals/debian-reference/)
1. [Debian 管理员手册](https://www.debian.org/doc/manuals/debian-handbook/index.zh-cn.html)
1. [第 5 章 网络设置](https://www.debian.org/doc/manuals/debian-reference/ch05.zh-cn.html)
