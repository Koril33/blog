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

本文简述对于简单脚本或者 web 服务的 unit 配置文件的编写和使用。

---

## systemd

systemd 通过 unit file 来管理服务，这些 unit file 本质是 ini 配置文件，它们定义了服务的详细信息，包括服务描述，如何启动、停止、重启以及服务之间的依赖关系。

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
3. [Install]: 定义 unit 的安装方式，服务是否能跟随系统启动有关。

### 示例

假设现在有一个 shell 脚本需要放到 systemd 中进行管理，那么可以编写如下 unit file：

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

现在如果执行了 test.service 的话，默认工作目录是根目录，demo.txt 会被写入到根目录下，同时，服务默认以 root 的身份执行的，此时 demo.txt 的属主和属组都是 root。

如果想要改变工作目录，以及更改执行命令时的用户，可以加入以下配置项：

```ini
[Unit]
Description=example systemd service unit file.

[Service]
# 改变工作目录，不指定的话，默认是根目录
WorkingDirectory=/home/bob
# 改变文件的属主和属组，不指定的话，默认是 root
User=bob
Group=bob
ExecStart=/bin/bash /usr/sbin/example.sh

[Install]
WantedBy=multi-user.target
```

添加了这三个配置项的作用是：工作目录改成 /home/bob，demo.txt 会被写入到工作目录下，并且执行脚本的用户和组是 bob，demo.txt 的属组和属主都变成了 bob。

## Service 中的 type

service 的 type 用于设置进程的启动类型。

type 可选值（simple, exec, forking, oneshot, dbus, notify, idle）解释如下：

- 如果设为 simple (当设置了 ExecStart= 、 但是没有设置 Type= 与 BusName= 时，这是默认值)， 那么 ExecStart= 进程就是该服务的主进程， 并且 systemd 会认为在创建了该服务的主服务进程之后，该服务就已经启动完成。 如果此进程需要为系统中的其他进程提供服务， 那么必须在该服务启动之前先建立好通信渠道(例如套接字)， 这样，在创建主服务进程之后、执行主服务进程之前，即可启动后继单元， 从而加快了后继单元的启动速度。 这就意味着对于 simple 类型的服务来说， 即使不能成功调用主服务进程(例如 User= 不存在、或者二进制可执行文件不存在)， systemctl start 也仍然会执行成功。
- exec 与 simple 类似，不同之处在于， 只有在该服务的主服务进程执行完成之后，systemd 才会认为该服务启动完成。 其他后继单元必须一直阻塞到这个时间点之后才能继续启动。换句话说， simple 表示当 fork() 函数返回时，即算是启动完成，而 exec 则表示仅在 fork() 与 execve() 函数都执行成功时，才算是启动完成。 这就意味着对于 exec 类型的服务来说， 如果不能成功调用主服务进程(例如 User= 不存在、或者二进制可执行文件不存在)， 那么 systemctl start 将会执行失败。
- 如果设为 forking ，那么表示 ExecStart= 进程将会在启动过程中使用 fork() 系统调用。 也就是当所有通信渠道都已建好、启动亦已成功之后，父进程将会退出，而子进程将作为主服务进程继续运行。 这是传统UNIX守护进程的经典做法。 在这种情况下，systemd 会认为在父进程退出之后，该服务就已经启动完成。 如果使用了此种类型，那么建议同时设置 PIDFile= 选项，以帮助 systemd 准确可靠的定位该服务的主进程。 systemd 将会在父进程退出之后 立即开始启动后继单元。
- oneshot 与 simple 类似，不同之处在于， 只有在该服务的主服务进程退出之后，systemd 才会认为该服务启动完成，才会开始启动后继单元。 此种类型的服务通常需要设置 RemainAfterExit= 选项。 当 Type= 与 ExecStart= 都没有设置时， Type=oneshot 就是默认值。
- dbus 与 simple 类似，不同之处在于， 该服务只有获得了 BusName= 指定的 D-Bus 名称之后，systemd 才会认为该服务启动完成，才会开始启动后继单元。 设为此类型相当于隐含的依赖于 dbus.socket 单元。 当设置了 BusName= 时， 此类型就是默认值。
- notify 与 exec 类似，不同之处在于， 该服务将会在启动完成之后通过 sd_notify(3) 之类的接口发送一个通知消息。systemd 将会在启动后继单元之前， 首先确保该进程已经成功的发送了这个消息。如果设为此类型，那么下文的 NotifyAccess= 将只能设为非 none 值。如果未设置 NotifyAccess= 选项、或者已经被明确设为 none ，那么将会被自动强制修改为 main 。注意，目前 Type=notify 尚不能与 PrivateNetwork=yes 一起使用。
- idle 与 simple 类似，不同之处在于， 服务进程将会被延迟到所有活动任务都完成之后再执行。 这样可以避免控制台上的状态信息与shell脚本的输出混杂在一起。 注意：(1)仅可用于改善控制台输出，切勿将其用于不同单元之间的排序工具； (2)延迟最多不超过5秒， 超时后将无条件的启动服务进程。

建议对长时间持续运行的服务尽可能使用 Type=simple (这是最简单和速度最快的选择)。

因为使用任何 simple 之外的类型都需要等待服务完成初始化，所以可能会减慢系统启动速度。 因此，应该尽可能避免使用 simple 之外的类型(除非必须)。另外，也不建议对长时间持续运行的服务使用 idle 或 oneshot 类型。

Type=simple，意味着脚本启动后，systemd 不会等待，立刻就会执行接下去的任务。

Type=oneshot，意味着 systemd 阻塞后续任务，直到脚本退出（无论成功或失败）。

如果脚本是初始化任务（如配置环境、挂载磁盘），且后续服务依赖其完成（如数据库需先初始化），必须用 oneshot 确保 systemd 等待脚本结束。

所以，一次性的脚本，建议使用 Type=oneshot，而 web 服务可以使用 Type=simple。

如果指定 Type=oneshot，一般还会增加一个参数，RemainAfterExit=yes，如果不加该参数，该参数默认值是 no，默认效果是：服务脚本执行完后变为 inactive（Active: inactive (dead)），看起来像“没有在运行”，加上 RemainAfterExit=yes 后，即使脚本退出，systemd 仍认为服务是 active (exited)（Active: active (exited)），方便依赖与判断任务是否完成。

---

## 简单脚本

简单脚本，以上配置文件修改如下：

```ini
[Unit]
Description=example systemd service unit file.

[Service]
# 一次性脚本
Type=oneshot
# 执行完毕后，状态保持 active
RemainAfterExit=yes
# 改变工作目录，不指定的话，默认是根目录
WorkingDirectory=/home/bob
# 改变文件的属主和属组，不指定的话，默认是 root
User=bob
Group=bob
ExecStart=/bin/bash /usr/sbin/example.sh

[Install]
WantedBy=multi-user.target
```

去掉注释版本，部分参数留空表示需要根据项目实际情况进行填写：

```ini
[Unit]
Description=

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=
User=
Group=
ExecStart=

[Install]
WantedBy=multi-user.target
```

---

## Web 服务的配置

实际工作中，大部分都是配置 web 服务，上面的示例仅仅适用于简单的脚本文件执行。部署 web 服务还需要其他的一些配置项。

### After 和 Wants

After 和 Wants 是用来表达依赖关系和启动顺序的，一般都会加入以下配置，因为 web 服务会依赖日志和网络服务：

```ini
[Unit]
After=network.target syslog.target
Wants=network.target
```

After 表示，这个服务要在 network.target 和 syslog.target 启动之后再启动，它只是表示启动顺序，并不意味着 systemd 会主动去启动这些目标（targets）或服务，只会等它们启动完成后才启动当前服务。

现代系统一般使用 journald 而不是 syslog.target，所以 syslog.target 可以去掉。

Wants 表示，当前服务希望 network.target 也能启动，这是一个弱依赖（weak dependency），如果 network.target 启动失败了，当前服务仍然会尝试启动。

### 关闭

我们在使用 systemctl stop 的时候，默认情况下，systemd 会向应用发送 SIGTERM，也就是 kill -15，也可以自定义 ExecStop：

```ini
[Service]
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
[Service]
RestartSec=5
```

这意味着系统会在重启服务前等待 5 秒钟。

[Unit] 中也可以指定 StartLimitBurst 和 StartLimitIntervalSec 来控制重启频率，防止重启次数过多。

```ini
[Unit]
StartLimitIntervalSec=60s
StartLimitBurst=3
```

这个配置的含义是：

1. 如果服务连续失败 3 次以内，systemd 会重启该服务。
2. 如果服务在 60 秒内失败超过 3 次，systemd 会停止重启，并把它标记为失败。

### 整合

把以上的配置项做一下整合，可以得到大部分 web 服务都可以使用的 service，假设 web 服务叫 demo-web.service：

```ini
[Unit]
# 描述信息
Description=The simple web server
# 启动顺序和依赖，这里可以使用 network.target
# 但是 network-online.target 更加精确的表明该 web 服务需要依赖网络已经完全可用的状态
After=network-online.target
Wants=network-online.target
# 控制重启频率
StartLimitIntervalSec=60s
StartLimitBurst=3

[Service]
# 显示设置 Type
Type=simple
# 改变工作目录，一般是 web 服务的目录，不指定的话，默认是根目录
WorkingDirectory=/home/bob/web
# 改变文件的属主和属组，不指定的话，默认是 root
User=bob
Group=bob
# 启动和停止命令
# Spring Boot 示例：ExecStart=/usr/bin/java -jar /opt/yourapp/app.jar
# Flask 示例: ExecStart=/usr/local/bin/gunicorn --workers 4 app:app
ExecStart=你的web服务的启动命令
ExecStop=/bin/kill -s TERM $MAINPID
# 什么情况下自动重启
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

去掉注释版本，部分参数留空表示需要根据项目实际情况进行填写：

```ini
[Unit]
Description=
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=60s
StartLimitBurst=3

[Service]
Type=simple
WorkingDirectory=
User=
Group=
ExecStart=
ExecStop=/bin/kill -s TERM $MAINPID
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## 资源限制

略

---

## journalctl

略

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
