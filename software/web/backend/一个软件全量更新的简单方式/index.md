---
title: "卸载和弹出移动硬盘"
date: 2025-05-01T16:41:00+08:00
summary: "如何在 Ubuntu 卸载并且弹出移动硬盘"
---

## 前言

之前开发过一些客户端小工具（CLI 或者 GUI），不可避免地会碰到软件更新的问题，如果是自己使用的软件，那用新的版本替换掉旧的版本即可，一切都在本地完成。

但如果开发的工具分发到互联网或者公司部门同事之间使用，软件的更新就成了一个头疼的问题，每次更新软件版本，都得通过聊天软件或者U盘之类的手段传来传去。

本文提供一个简单通用的方式（不一定是最好的），来实现软件的全量更新。

---

## 思路

用户启动软件后，一般可以通过以下几种方式来提供更新入口：

1. 启动软件时，立即检测是否有新的版本，用户被动接收更新通知
2. 软件启动后，定时检测是否有新的版本，用户被动接收更新通知
3. 菜单栏的帮助里，一般会提供检测是否有新版本的选项，用户主动检查更新

无论哪一种更新方式，首先需要实现的函数是检测是否存在新的版本。

可以在公网服务器上，放置一个文本文件（格式随意，txt，yaml，json，xml 都行），该文本文件存储了当前最新版本的软件信息:

- 新版本的版本号
- 新版本的发布时间
- 更新内容
- 新版本的下载地址

检测是否存在新的版本，就可以通过该文件的最新版本号和当前客户端软件内部的版本号进行比对。如果存在更新的版本，则根据该文件提供的新版本的下载地址，下载到本地的临时目录。

新版本的软件下载到临时目录成功后，再进行替换，把旧版本替换成新版本，然后重新启动软件，并且退出更新程序。

需要注意的是，网上的一些方案，是把更新程序和主程序捆绑，最后打包成一个 exe 文件，但是这样会导致“关闭主程序-更新-重启主程序”这个主要环节出现问题，Windows 是不能关闭当前正在运行的程序的，所以一些博客的方法是把“软件替换+重启软件”这个步骤通过 bat 文件来完成。

本文的方式是把 app.exe（主程序）和 update.exe（更新程序）分开，这样“关闭-下载-替换-重启”的整个步骤会更加方便。

---

## 主程序

主程序不包含其他业务逻辑，仅仅把当前软件的版本号显示在窗口中央：

```python
import os
import subprocess
import sys

from PySide6.QtCore import Qt
from PySide6.QtWidgets import (
    QMainWindow, QApplication, QLabel,
    QMessageBox
)

__version__ = 'v1.0.0'


class MainWindow(QMainWindow):
    def __init__(self):
        super().__init__()
        self.label = None
        self.init_ui()

    def init_ui(self):
        self.setWindowTitle("Hello World")
        self.setFixedSize(300, 150)

        self.label = QLabel(f"version: {__version__}", self)
        self.label.setAlignment(Qt.AlignmentFlag.AlignCenter)
        self.setCentralWidget(self.label)


def main():
    app = QApplication(sys.argv)
    window = MainWindow()
    window.show()
    sys.exit(app.exec())


if __name__ == '__main__':
    main()
```

这里采用的更新方式是，在菜单栏放一个 help，提供更新的选项：

!()[./images/1.jpg]

菜单栏代码如下：

```python
def init_ui(self):

    # 添加菜单
    menubar = self.menuBar()
    update_menu = menubar.addMenu("Help")
    check_action = update_menu.addAction("Check App Update")
    check_action.triggered.connect(self.check_update)
```

用户点击 check action 后，回调 check_update 函数：

```python
    def check_update(self):
        try:
            version_info = update.check_version(__version__)
            if version_info:
                reply = QMessageBox.question(
                    self,
                    "发现新版本",
                    f"检测到新版本 {version_info['version']}，是否更新？",
                    QMessageBox.StandardButton.Yes | QMessageBox.StandardButton.No
                )
                if reply == QMessageBox.StandardButton.Yes:
                    self.run_update(version_info)
            else:
                QMessageBox.information(self, "已是最新版", f"当前已是最新版本：{__version__}")
        except Exception as e:
            QMessageBox.critical(self, "更新错误", f"检查更新时出错: {str(e)}")
```

update.check_version 会去检测是否存在新的版本，如果存在的话，就把服务器上的版本信息返回。

用户点击确认更新后，调用 run_update 函数，该函数主要作用是启动更新程序并且退出当前主程序：

```python
    def run_update(self, version_info):
        try:

            app_path = Path(sys.argv[0])
            app_dir = app_path.parent

            # 构建 update.exe 路径并检查存在性
            update_path = app_dir / "update.exe"
            if not update_path.exists():
                raise FileNotFoundError(f"找不到更新程序: {update_path}")

            # 启动更新程序
            subprocess.Popen([
                update_path,
                app_path,
                __version__,
                version_info['exe_url'],
                version_info['version'],
                version_info['description'],
                version_info['update_date']
            ])

            # 退出当前应用
            QApplication.quit()
        except Exception as e:
            QMessageBox.critical(self, "更新错误", f"启动更新程序时出错: {str(e)}")
```


