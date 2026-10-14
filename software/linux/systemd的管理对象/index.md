---
title: systemd的管理对象
date: '2026-10-14T11:14:50+08:00'
summary: 'systemd的所有unit含义与作用'
draft: false
---

## unit

systemd 管理的一切对象都叫 **unit（单元）**，每个 unit 对应一个配置文件，文件后缀表示类型。

| 类型              | 作用                                                   | 场景                                                           |
| ----------------- | ------------------------------------------------------ | -------------------------------------------------------------- |
| .service          | 管理一个进程/服务，通常是后台长期运行的服务            | Nginx、sshd、自己编写的后台服务                                |
| .timer            | 基于时间的触发器(类似 cron)，触发关联的unit            | 取代传统的 cron，比如数据备份                                  |
| .socket           | 监听套接字，有连接时再拉起服务                         | docker.socket、sshd.socket、systemd-journald.socket            |
| .target           | 聚合一组 unit 的集合，相当于"状态"，启动阶段的"里程碑" | multi-user.target、graphical.target、network-online.target     |
| .mount/.automount | 管理挂载点                                             | 挂载 NFS 共享、automount 避免网络盘拖慢开机                    |
| .path             | 基于路径的触发器，监视文件/目录变化并触发 unit         | 上传目录有文件就处理，配置文件变动后自动重载服务               |
| .slice/.scope     | 资源控制（cgroup）分组，限制 CPU/内存                  | 限制一组批处理任务总资源，限制某个用户的资源，临时限制某条命令 |
| .device/.swap     | 设备、交换分区，一般由系统自动生成，不手写             | 设备插入后才启动服务，用 udev 规则触发服务                     |

对于日常运维开发，主要了解 .service，.timer，.target，.socket，其余用到时再查阅。
