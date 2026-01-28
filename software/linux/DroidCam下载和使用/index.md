---
title: "DroidCam下载和使用"
date: 2026-01-28T23:13:25
summary: ""
---

## DroidCam

DroidCam 的主要作用是把手机摄像头可以当作 Web Camera，本文用 USB 连接手机和电脑的方式来简单介绍下 DroidCam 的使用。

## 准备

手机打开开发者模式，并且设置允许 USB debug，电脑安装好 adb：

```sh
sudo apt install -y adb
```

下载好 v4l2loopback 模块，该模块的目的是为了创建虚拟视频设备，方便视频流供其他软件（OBS、ZOOM、OpenCV）调用。

```sh
sudo apt install linux-headers-$(uname -r) v4l2loopback-dkms
```

---

## 下载

手机安装好 DroidCam APP，电脑端则去 Github release 下载客户端：https://github.com/dev47apps/droidcam-linux-client/releases

解压后，执行：

```sh
koril@DjhLaptop:/opt/droidcam$ sudo ./install-client 
Copying files
+ cp uninstall /opt/droidcam-uninstall
+ cp icon2.png /opt/droidcam-icon.png
+ cp droidcam /usr/local/bin/
+ cp droidcam-cli /usr/local/bin/
+ set +x
Done
```

---

## 使用

adb 确保手机 USB 正常连接：

```sh
koril@DjhLaptop:/opt/droidcam$ sudo adb devices 
List of devices attached
2121156a	device

```

加载 v4l2loopback 内核模块，并且手动创建 1 个虚拟摄像头，名字叫 DroidCam，设备文件是 /dev/video5：

```sh
sudo modprobe v4l2loopback devices=1 video_nr=5 card_label="DroidCam"
```

查看 video 设备，应该会有个 video5：

```sh
koril@DjhLaptop:/opt/droidcam$ ls -l /dev/video*
crw-rw----+ 1 root video 81, 0 Jan 28 22:16 /dev/video0
crw-rw----+ 1 root video 81, 1 Jan 28 22:16 /dev/video1
crw-rw----+ 1 root video 81, 2 Jan 28 22:16 /dev/video2
crw-rw----+ 1 root video 81, 3 Jan 28 22:16 /dev/video3
crw-rw----+ 1 root video 81, 4 Jan 28 22:40 /dev/video5
```

打开手机 DroidCam APP，电脑端执行：

```sh
koril@DjhLaptop:/opt/droidcam$ sudo droidcam-cli adb 4747
Audio loopback device not found.
Is snd_aloop loaded?
Client v2.1.4
Video: /dev/video5
```

手机摄像头会被拉起，现在视频流就被写入到了`/dev/video5`中。

## opencv

可以写一个脚本测试：

```sh
import cv2

# /dev/video5 → index = 5
cap = cv2.VideoCapture(5, cv2.CAP_V4L2)

if not cap.isOpened():
    print("❌ 无法打开摄像头")
    exit(1)

while True:
    ret, frame = cap.read()
    if not ret:
        print("❌ 读取失败")
        break

    cv2.imshow("DroidCam (Phone Camera)", frame)

    # 按 ESC 退出
    if cv2.waitKey(1) & 0xFF == 27:
        break

cap.release()
cv2.destroyAllWindows()
```

---

## 参考

1. https://droidcam.app/
2. https://github.com/v4l2loopback/v4l2loopback
3. https://github.com/dev47apps/droidcam-linux-client/

