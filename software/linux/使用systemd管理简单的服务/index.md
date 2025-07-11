---
title: "使用systemd管理简单的服务"
date: 2025-07-11T15:22:00+08:00
summary: "关于 Unit file 的编写，systemctl 和 journalctl 的使用说明"
---

## 目录

[TOC]

---

## 前言

大部分的 Linux 发行版现在都采用了 systemd 系统来启动和管理各种服务。

本文简述对于简单脚本或者服务的 unit 配置文件的编写和使用。

---

## systemd

systemd 通过 unit file 来管理服务，这些 unit file 定义了服务的详细信息，包括服务描述，如何启动、停止、重启以及服务之间的依赖关系。

unit file 通常存储在 /etc/systemd/system/ （用户定义的服务）或 /lib/systemd/system/ （系统服务）中。

---

## Unit文件

systemd 的 unit 有很多种类，根据后缀名能够进行区分，有以下种类：

- .service
- .scope
- .path
- .snapshot
- .timer
- .slice
- .target
- .socket
- .device
- .swap
- .mount
- .automount

本文讲的是最常见的 service 服务。

unit file 就是一个 ini 格式的配置文件，主要包含 3 个 section

1. [Unit]: 描述和定义 unit 的一些元数据信息。
2. [Service]: 定义 unit 如何启动，关闭，重启等。
3. [Install]: 定义 unit 的安装方式，和服务是否能跟随系统启动有关。

### 示例

假设现在有一个脚本需要放到 systemd 中进行管理，那么可以编写如下 unit file：

```ini
[Unit]
Description=example systemd service unit file.

[Service]
ExecStart=/bin/bash /usr/sbin/example.sh

[Install]
WantedBy=multi-user.target
```

unit file 路径：/etc/systemd/system/test.service

[Unit] 的 Description 是描述该服务的内容，编写时应该尽可能简洁明了，一眼能看懂该服务的主要功能。

[Service] 的 ExecStart 用于指定如何启动该服务，这里是一个简单的脚本——使用 bash 执行一个 sh 文件，尽可能使用绝对路径。

[Install] 的 WangtedBy 表示当启用（enable）该服务时，要将其加入到哪些 target 的 wants 目录中。

example.sh 的内容如下：

```sh
echo "this is a test" > demo.txt
```
---

## systemctl

准备好了 service 文件和 sh 脚本文件之后，需要告知 systemd 加入了新的服务——test.service，使用以下命令：

```sh
sudo systemctl daemon-reload
```

启动该服务：

```sh
sudo systemctl start test.service
```

加入开机自启：

```sh
sudo systemctl enable test.service
```

---

## 工作目录和用户

现在如果执行了 test.service 的话，默认工作目录是根目录，demo.txt 会被写入到根目录下，同时，服务默认是以 root 的身份执行的。

如果想要改变工作目录，以及执行用户，可以加入以下配置项：

```ini
[Unit]
Description=example systemd service unit file.

[Service]
WorkingDirectory=/home/bob
User=bob
Group=bob
ExecStart=/bin/bash /usr/sbin/example.sh

[Install]
WantedBy=multi-user.target
```

添加了这三个配置项的作用是：工作目录改成 /home/bob，demo.txt 会被写入到工作目录下，并且执行脚本的用户和组是 bob，demo.txt 的属组和属主都变成了 bob。

---

## Web 服务相关的配置

实际工作中，大部分都是配置 web 服务，上面的示例仅仅适用于简单的脚本文件执行。部署 web 服务还需要其他的一些配置项。

### After 和 Wants

After 和 Wants 是用来表达依赖关系和启动顺序的，一般都会加入以下配置，因为 web 服务会依赖日志和网络服务：

```init
[Unit]
After=network.target syslog.target
Wants=network.target
```

After 表示，这个服务要在 network.target 和 syslog.target 启动之后再启动，它只是表示启动顺序，并不意味着 systemd 会主动去启动这些目标（targets）或服务，只会等它们启动完成后才启动当前服务。

Wants 表示，当前服务“希望”network.target 也能启动，这是一个弱依赖（weak dependency），如果 network.target 启动失败了，当前服务仍然会尝试启动。

### 关闭

我们在使用 systemctl stop 的时候，默认情况下，systemd 会向应用发送 SIGTERM，也就是 kill -15，也可以自定义 ExecStop：

```ini
ExecStop=/bin/kill -s TERM $MAINPID
```

### 自动重启

systemd 默认不会自动重启服务，可以通过 Restart 参数指定在什么情况下进行自动重启，可选值如下：

- no: 默认值，不重启
- on-success: 服务进程正常退出时也会重启（不常用），退出码为 “0”，或者进程收到 SIGHUP，SIGINT，SIGTERM，SIGPIPE。
- on-failure: 仅在服务进程异常退出（退出码不为“0”，或者进程被强制杀死，或者 systemd 操作超时）时，会重启。
- always: 无条件重启

除此之外，还可以指定 RestartSec 参数，表明服务应该多久后进行重启，可以避免过快的重启。

```ini
RestartSec=5
```

这意味着系统会在重启服务前等待 5 秒钟。

[Unit] 中也可以指定 StartLimitBurst 和 StartLimitIntervalSec 来控制重启频率

```ini
[Unit]
StartLimitInterval=60s
StartLimitBurst=3
```

这个配置的含义是：

1. 如果服务连续失败 3 次以内，systemd 会重启该服务。
2. 如果服务在 60 秒内失败超过 3 次，systemd 会停止重启，并把它标记为失败。

---

## journalctl

---

## 参考

1. https://www.jinbuguo.com/systemd/systemd.index.html
2. https://docs.redhat.com/zh-cn/documentation/red_hat_enterprise_linux/8/html/using_systemd_unit_files_to_customize_and_optimize_your_system/index
3. https://www.siberoloji.com/how-to-manage-systemd-services-in-debian-12-bookworm/
4. https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files
5. https://www.suse.com/support/kb/doc/?id=000019672
6. https://gofrp.org/zh-cn/docs/setup/systemd/
7. https://documentation.suse.com/smart/systems-management/html/systemd-setting-up-service/index.html
8. https://0pointer.de/blog/projects/systemd.html
9. https://zhuanlan.zhihu.com/p/271071439
10. https://blog.liaosirui.com/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6/B.%E8%AE%A1%E7%AE%97%E6%9C%BA%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/Linux/Systemd/Systemd%E5%B7%A5%E5%85%B7%E9%9B%86.html
11. https://webdock.io/en/docs/how-guides/system-maintenance/quick-guide-managing-systemd-services?srsltid=AfmBOoqXjhBEtWLqp1GWfhjVuV1cUTMzsaK4rLnLc3lQs2ItxVh7xCuv
12. https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html
13. https://www.redhat.com/en/blog/systemd-automate-recovery
