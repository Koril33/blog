---
title: Linux用户被删除后uid的归属问题
date: '2026-10-22T17:37:44+08:00'
summary: '后来新建的用户可能会继承之前的，已被删除的用户的权限'
draft: false
---

## UID 复用

考虑以下的场景，系统中新增一个用户 ex1，假设它的 uid=1001：

```shell
sudo useradd -m ex1

grep ex1 /etc/passwd
ex1:x:1001:1001::/home/ex1:/bin/sh
```

在 /var 下面新建一个文件：

```shell
sudo -u ex1 touch /var/tmp/owned
```

这个文件的属主和属组都是 ex1：

```shell
ls -lA /var/tmp/owned 
-rw-r--r-- 1 ex1 ex1 0 Oct 22 17:20 /var/tmp/owned
```

删除用户 ex1：

```shell
sudo userdel ex1
```

虽然 ex1 被删除了，但是它相关的文件并没有自动删除，而且属组和属主都变成了数字1001：

```shell
ls -lA /var/tmp/owned 
-rw-r--r-- 1 1001 1001 0 Oct 22 17:20 /var/tmp/owned
```

接下来再次创建一个新的用户——ex2：

```shell
sudo useradd ex2; id -u ex2
1001
```

会发现 ex2 的 uid 也是 1001，也就是说 1001 这个 uid 被复用了。

原来属于 ex1 的目录或者文件会自动被 ex2 继承：

```shell
ls -lA /home
total 8
drwxr-xr-x 2 ex2   ex2   4096 Oct 22 17:20 ex1

ls -lA /var/tmp/owned 
-rw-r--r-- 1 ex2 ex2 0 Oct 22 17:20 /var/tmp/owned
```

---

## 用户和 uid

从系统层面来看，用户的本质就是 uid 这个数字，系统只判断数字，而不关心数字关联的用户名，root 的特殊在于 **uid 为 0**，不在名字。内核判定特权看的是 uid 0 和 capabilities，从不比较字符串“root”。

`useradd` 分配 uid 时，是在 `UID_MIN`～`UID_MAX`（`/etc/login.defs`）范围内取**比当前已存在的最大 uid 更大的下一个值**；如果已经到上限，才回头找空闲值。所以删掉的恰好是最大的那个 uid（比如 ex1 是 1001 且是最大的），下一个新用户就会重新拿到 1001。如果中间还有别的更大 uid 的用户，通常不会复用。

uid 分段（Debian Policy 与 `/etc/login.defs`）：

| 范围        | 含义                                                                 |
| ----------- |:--------------------------------------------------------------------:|
| 0           | root                                                                 |
| 1–99        | 发行版静态分配的系统账户                                             |
| 100–999     | 动态分配的系统用户（`SYS_UID_MIN/MAX`），`useradd --system` 落在这里 |
| 1000–60000  | 普通用户（`UID_MIN/UID_MAX`）                                        |
| 60000–64999 | Debian 保留，全局分配                                                |
| 61184–65519 | systemd 的 `DynamicUser=` 使用                                       |
| 65534       | `nobody`                                                             |

---

## 解决方案

### 删除前先清点并处理文件

找到待删除用户uid相关的文件，进行处理：

```shell
# 找出所有属于该 uid 的文件（跨文件系统边界要小心）
sudo find / -xdev -uid 1001 -ls 2>/dev/null

# 选择：删除、归档，或转交给别人
sudo find / -xdev -uid 1001 -exec chown nobody:nogroup {} +   # 转交
# 或者归档后删除
```

然后再 `userdel -r ex1`（`-r` 会删家目录和邮件池，但不会删散落在其他位置的文件，所以上面的 `find` 不能省）。

### 不删除，改为锁定/禁用账户

```shell
sudo usermod -L -e 1 -s /usr/sbin/nologin ex1
# -L 锁密码，-e 1 让账户过期，-s 禁止登录 shell
```

账户条目还在 `/etc/passwd`，uid 就永远不会被分配给别人。离职员工、停用服务账户都适合这样做。

---

## 参考

1. https://www.php.cn/faq/2894175.html
