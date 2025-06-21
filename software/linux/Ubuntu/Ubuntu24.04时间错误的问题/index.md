---
title: "Ubuntu24.04时间错误的问题"
date: 2025-06-17T20:10:00+08:00
summary: "Ubuntu时间错误的解决方法"
---

## 问题

突然发现 Ubuntu 的时间错误了：

```sh
timedatectl status

               Local time: Wed 2025-06-18 03:51:53 CST
           Universal time: Tue 2025-06-17 19:51:53 UTC
                 RTC time: Tue 2025-06-17 19:51:52
                Time zone: Asia/Shanghai (CST, +0800)
System clock synchronized: no
              NTP service: active
          RTC in local TZ: no
```

可以看到时区是正确的，但是 RTC/UTC/Loca 时间都错了，Local time 应该是 2025-06-17 19:51:53，也就是说，Local time 比正确 UTC 时间超了 16 小时。

查看了下 systemd-timesyncd.service 的日志：

```sh
koril@ThinkBook:~$ systemctl status systemd-timesyncd.service
● systemd-timesyncd.service - Network Time Synchronization
     Loaded: loaded (/usr/lib/systemd/system/systemd-timesyncd.service; enabled>
     Active: active (running) since Wed 2025-06-18 03:45:24 CST; 8min ago
       Docs: man:systemd-timesyncd.service(8)
   Main PID: 6904 (systemd-timesyn)
     Status: "Idle."
      Tasks: 2 (limit: 37806)
     Memory: 1.4M (peak: 2.1M)
        CPU: 35ms
     CGroup: /system.slice/systemd-timesyncd.service
             └─6904 /usr/lib/systemd/systemd-timesyncd

Jun 18 03:47:08 ThinkBook systemd-timesyncd[6904]: Timed out waiting for reply >
Jun 18 03:47:18 ThinkBook systemd-timesyncd[6904]: Timed out waiting for reply >
Jun 18 03:48:33 ThinkBook systemd-timesyncd[6904]: Timed out waiting for reply >
Jun 18 03:48:43 ThinkBook systemd-timesyncd[6904]: Timed out waiting for reply >
Jun 18 03:48:53 ThinkBook systemd-timesyncd[6904]: Timed out waiting for reply >
Jun 18 03:49:04 ThinkBook systemd-timesyncd[6904]: Timed out waiting for reply >
Jun 18 03:51:22 ThinkBook systemd-timesyncd[6904]: Timed out waiting for reply >
Jun 18 03:51:32 ThinkBook systemd-timesyncd[6904]: Timed out waiting for reply >
Jun 18 03:51:43 ThinkBook systemd-timesyncd[6904]: Timed out waiting for reply >
Jun 18 03:51:53 ThinkBook systemd-timesyncd[6904]: Timed out waiting for reply >

```

看到**Timed out waiting for reply**就大概知道了，估计还是网络（墙）的问题。

---

## 解决

修改配置文件，指定国内的 ntp 服务器

```sh
# 备份
sudo cp /etc/systemd/timesyncd.conf /etc/systemd/timesyncd.conf.bak
# 修改
sudo vim /etc/systemd/timesyncd.conf
```

修改内容如下：
```sh
[Time]
NTP=ntp.tencent.com
FallbackNTP=ntp1.tencent.com,ntp2.tencent.com,ntp3.tencent.com
RootDistanceMaxSec=5
PollIntervalMinSec=32
PollIntervalMaxSec=2048
```

这里可以自行搜索可靠的国内 ntp 服务器。

重启 systemd-timesyncd.service：

```sh
# 重启
sudo systemctl restart systemd-timesyncd.service
# 启用网络时间同步
sudo timedatectl set-ntp on
```

---

## 参考

1. https://plumz.me/archives/13983/
2. https://developer.aliyun.com/article/1141208
