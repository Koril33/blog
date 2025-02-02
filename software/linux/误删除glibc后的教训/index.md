---
title: "误删除glibc后的教训"
date: 2023-07-20T16:04:55+08:00
tags: []
featured_image: "images/background.jpg"
summary: "造成过的最大的事故"
toc: true
---

## 前言

前两天，在一台线上服务器准备安装 gcc 的时候，发现 glibc 包版本过低，遂天真的执行了 rpm -e --nodeps glibc，然后发现 Linux 基本命令除了 cd 之外几乎都用不了了。

客观原因是，glibc 是整个 Linux 系统底层的依赖库，一旦被删除，就会爆出 /lib64/ld-linux-x86-64.so.2: bad ELF interpreter: No such file or directory 的错误。

主观原因是，对 rpm 和 glibc 的了解不够深入，过于草率。

---

## 解决方案

我的解决方案是，上传 [busybox](https://busybox.net/downloads/binaries/1.17.2/busybox-x86_64) 到服务器上，然后通过 Winscp 添加了 busybox 的执行权限，最后通过 busybox rpm -ivh glibc-2.17-317.el7.x86_64.rpm 将 glibc 重新装了回来，恢复成功。

这里能成功恢复，有四个点：

1. 能上传静态编译的 busybox 到服务器。
2. ssh 和 文件上传的连接没有断开（一旦重启或者 ssh 断开了，就彻底没法操作了，只能跑到物理机旁边了）。
3. 可以给 busybox 可执行权限。
4. 此时登录的是 root 用户。

一开始，我并没有想到能通过 winscp 可视化界面设置文件的执行权限（因为此时 chmod +x 也已经用不了了），所以搜到了下文：

https://zhuanlan.zhihu.com/p/20062978

思路是，cp 命令此时拥有执行位，所以将 busybox 的二进制内容写入到 /bin/cp，就像下面这样：

```shell
printf 'busybox 的所有二进制内容' > /bin/cp
```

问题是 busybox 有 898 KB，ssh 工具的命令行界面肯定没法一下子粘贴这么多内容（极容易挂掉 ssh 连接，而我们有仅有一次机会）。

所以，我还写了以下代码，去分割 busybox，每个文件 10000 字节，有 91 个文件，也就是说至少要复制粘贴 91 次。

```java
package cn.korilweb;

import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;

public class Main {
    public static void main(String[] args) throws IOException {
        Path p1 = Paths.get("C:\\users\\dingj\\desktop\\busybox-x86_64");
        Path p2 = Paths.get("C:\\users\\dingj\\desktop\\test\\");
        int len = 10000;

        byte[] buf = new byte[len];
        try (InputStream inputStream = Files.newInputStream(p1)) {
            int offset = 0;
            int cnt = 0;
            int readLen;
            while ((readLen = inputStream.read(buf, offset, len)) != -1) {
                OutputStream outputStream = Files.newOutputStream(p2.resolve(String.valueOf(cnt)));
                outputStream.write(buf, 0, readLen);
                cnt++;
            }
        }
    }
}
```

就在这时候，同事提醒我能够用 winscp 直接改可执行权限，才免了这些复杂的操作。

下面这个贴子的操作和我的思路差不多，不过他的 ssh 连接最后还是断开了：

https://www.v2ex.com/t/356113

---

## 经验教训

1. 谨慎操作 Linux 服务器。
2. 了解 rpm 和其他安装方式的种种细节。
