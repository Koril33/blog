---
title: "卸载和弹出移动硬盘"
date: 2025-05-01T16:41:00+08:00
summary: "如何在 Ubuntu 卸载并且弹出移动硬盘"
---

## 查看

查看移动硬盘有几个相关的命令：

```
sudo fdisk -l

lsblk

df -h
```

我本地挂载了移动硬盘，在这两个命令中，显示的是：

```shell
sudo fdisk -l 的显示结果
Device     Boot Start        End    Sectors  Size Id Type
/dev/sda1  *     2048 3907027119 3907025072  1.8T  7 HPFS/NTFS/exFAT


lsblk 的显示结果
sda           8:0    0   1.8T  0 disk 
└─sda1        8:1    0   1.8T  0 part /media/koril/TOSHIBA EXT

df -h 的显示结果
/dev/sda1       1.9T  1.7T  177G  91% /media/koril/TOSHIBA EXT
```

我的用户名是 koril，所以系统自动挂载到的路径是：

```shell
$ ls -ld /media/koril/TOSHIBA\ EXT/
drwxrwxrwx 1 koril koril 8192 Apr 12 01:56 '/media/koril/TOSHIBA EXT/'
```

---

## 卸载

卸载的命令，会让系统无法访问该分区，但实际上不会断电：

```shell
$ udisksctl unmount -b /dev/sda1
Unmounted /dev/sda1.
```

这个命令和 umount 类似，但是 umount 是需要 sudo 权限的。执行这个命令后，USB 设备保持物理连接，硬盘仍可重新挂载（通过 mount 或者 udisksctl mount 指令）。

---

## 断电

虽然卸载了设备，但是指示灯还是亮着的，在卸载的基础上，进一步对设备进行物理断电：

```shell
$ udisksctl power-off -b /dev/sda
```

这个命令向设备发送“安全断电”指令（如果设备支持），彻底关闭闭硬盘电源，设备不再转动（适用于外置 USB 硬盘或 SSD）。想要重新挂载，需要重新连接或重新上电。

卸载的时候指定的是 /dev/sda1 表示分区，关闭电源的时候指定的是 /dev/sda 表示整个硬盘。

需要注意，power-off 的目标是对整个物理设备（如整个磁盘 /dev/sda`）断电，而非单个分区（如 /dev/sda1）。若设备上有任何分区仍处于挂载状态，操作会失败，对磁盘的断电操作需确保其所有子设备（分区）均已处于非活跃状态。

在确定了分区和磁盘的时候，可以直接一行命令卸载+断电：

```shell
$ udisksctl unmount -b /dev/sda1 && udisksctl power-off -b /dev/sda 
```

