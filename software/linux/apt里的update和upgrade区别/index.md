---
title: apt里的update和upgrade区别
date: '2026-10-22T16:05:38+08:00'
summary: '为什么需要先 update 再 upgrade'
draft: false
---

## update

apt update 只是从软件源下载最新的软件包索引（哪些软件有新版本、版本号是多少），不会安装或升级任何软件。

就像手机应用商店刷新页面，可以知道哪些 App 有更新，但还没点"更新"。

```shell
alice@computer:~$ sudo apt update
[sudo] password for alice: 
Hit:1 http://mirrors.cloud.aliyuncs.com/debian bookworm InRelease
Get:2 http://mirrors.cloud.aliyuncs.com/debian-security bookworm-security InRelease [34.8 kB]
Get:3 http://mirrors.cloud.aliyuncs.com/debian bookworm-updates InRelease [55.4 kB]
Get:4 http://mirrors.cloud.aliyuncs.com/debian bookworm-backports InRelease [59.4 kB]
Get:5 http://mirrors.cloud.aliyuncs.com/debian-security bookworm-security/main Sources [277 kB]
Get:6 http://mirrors.cloud.aliyuncs.com/debian-security bookworm-security/main amd64 Packages [355 kB]
Get:7 http://mirrors.cloud.aliyuncs.com/debian-security bookworm-security/main Translation-en [221 kB]       
Get:8 http://mirrors.cloud.aliyuncs.com/debian bookworm-backports/main Sources.diff/Index [63.3 kB]          
Get:9 http://mirrors.cloud.aliyuncs.com/debian bookworm-backports/main amd64 Packages.diff/Index [63.3 kB]
Get:10 http://mirrors.cloud.aliyuncs.com/debian bookworm-backports/main Translation-en.diff/Index [63.3 kB]
Get:11 http://mirrors.cloud.aliyuncs.com/debian bookworm-backports/main Sources T-2026-10-03-2010.25-F-2026-10-03-2010.25.pdiff [118 B]
Get:11 http://mirrors.cloud.aliyuncs.com/debian bookworm-backports/main Sources T-2026-10-03-2010.25-F-2026-10-03-2010.25.pdiff [118 B]
Get:12 http://mirrors.cloud.aliyuncs.com/debian bookworm-backports/main amd64 Packages T-2026-10-03-2010.25-F-2026-10-03-2010.25.pdiff [87 B]
Get:12 http://mirrors.cloud.aliyuncs.com/debian bookworm-backports/main amd64 Packages T-2026-10-03-2010.25-F-2026-10-03-2010.25.pdiff [87 B]
Get:13 http://mirrors.cloud.aliyuncs.com/debian bookworm-backports/main Translation-en T-2026-10-03-2010.25-F-2026-10-03-2010.25.pdiff [62 B]
Get:13 http://mirrors.cloud.aliyuncs.com/debian bookworm-backports/main Translation-en T-2026-10-03-2010.25-F-2026-10-03-2010.25.pdiff [62 B]
Get:14 https://artifacts.elastic.co/packages/8.x/apt stable InRelease [3,249 B]                             
Get:15 https://artifacts.elastic.co/packages/8.x/apt stable/main amd64 Packages [115 kB]
Fetched 1,312 kB in 2s (681 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
75 packages can be upgraded. Run 'apt list --upgradable' to see them.
alice@computer:~$ 

```

---

## upgrade

apt upgrade 则是根据刚刚 apt update 得到的更新清单，把已安装的软件升级到最新版本。

apt upgrade 只升级，不会删除任何已安装的软件，也不会为了满足依赖而移除旧包。如果某个升级需要新增或删除包才能完成，它会跳过这个包，显示"保留不升级"。

---

## 执行顺序

upgrade 依赖本地的索引来判断有没有新版本。不先 update，本地索引是旧的，它就不知道有新版本，等于白跑。所以标准流程是：

```shell
sudo apt update && sudo apt upgrade -y
```

---

## full-upgrade

比 upgrade 更加激进，可以为了完成升级而删除现有软件包，适用于内核、底层库等依赖关系变化较大的更新。执行前应检查将要删除的软件包。

---

## 更新频率

apt update 只刷新索引，没有任何风险，想跑就跑。通常是升级前顺带执行，或者用定时任务每天自动跑一次，系统默认的 `apt-daily` 定时器就是这么做的：

```shell
systemctl list-timers 'apt-daily*'
```

出现高危漏洞（CVE），尤其是影响你正在用的组件：`openssh`、`openssl`、`nginx`、`sudo`、内核等，应立即或 24~72 小时内进行 apt upgrade。

---

## 需要特别关注的包

生产环境下，有些包不能轻易更新：

- 数据库（`mysql`、`postgresql`）：版本升级可能涉及数据格式
- `nginx` / `apache`：配置兼容性
- 内核 / `glibc`：需要重启
- 运行环境（`php`、`python`、`nodejs`）：可能破坏应用依赖
