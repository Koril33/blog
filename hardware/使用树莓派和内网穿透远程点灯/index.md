---
title: "使用树莓派和内网穿透远程点灯"
date: 2021-06-15T17:44:58+08:00
summary: "树莓派+frp+SpringBoot 远程点亮 LED"
featured_image: "images/background.jpg"
toc: true
---

## 简介

树莓派作为web服务器，同时通过 GPIO 口控制 LED 灯泡的亮灭，如果要在公网ip进行访问控制，需要借助内网穿透的技术，这里选用 frp 进行穿透。

---

## 目录

[TOC]

---

## 准备

材料：

* 树莓派4b
* 腾讯云服务器
* LED 灯珠

---

## 树莓派安装Java 11


运行以下命令安装最新的 JDK 版本，目前是 OpenJDK 11 JDK：

```
sudo apt update
sudo apt install default-jdk
```

安装完成后，通过命令可以检查 Java 版本进行验证：

```
java -version
```

---

## 升级WiringPi

WiringPi预装（Pre-installed）在标准的树莓派操作系统Raspbin中。可以使用下面的命令进行安装：

```
sudo apt-get install wiringpi
```

如果需要更新WiringPi，可以使用系统更新命令：

```
sudo apt-get update
sudo apt-get upgrade
```

WiringPi安装完成后，可以使用下面的命令测试是否安装成功：

```
sudo gpio -v
```

另外，可以通过命令打印树莓派引脚分布图

```
gpio readall
```

---

## 内网穿透配置

腾讯云服务端配置：

```ini
[common]
bind_port = 7000

vhost_http_port = 8090
vhost_https_port = 443

[web]
type = http
custom_domains = korilweb.cn
```

这里设置 http port 为8090，相当于把这个端口和树莓派指定的端口进行绑定。

树莓派客户端配置：

```ini
[common]
server_addr = 106.54.131.221
server_port = 7000

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6000

[web]
type = http
local_ip = 127.0.0.1
local_port = 9000
custom_domains = korilweb.cn
```

custom_domains 必须和服务端的一致。

---

## 树莓派的web服务器

这里采用 Springboot + pi4j 的方案，Springboot 处理 web 请求，pi4j 处理 GPIO。

### 引入 pi4j

在 pom.xml 中添加配置

```xml
<!-- https://mvnrepository.com/artifact/com.pi4j/pi4j-core -->
<dependency>
    <groupId>com.pi4j</groupId>
    <artifactId>pi4j-core</artifactId>
    <version>1.4</version>
</dependency>
```

### Controller

```java
@RestController
public class HelloController {

    final GpioController gpio = GpioFactory.getInstance();
    final GpioPinDigitalOutput led1 = gpio.provisionDigitalOutputPin(RaspiPin.GPIO_01);
    @GetMapping(value = "/command/{status}")
    public String test(@PathVariable String status) {
        if (status.equals("open")) {
            led1.high();
            return "led open";
        }
        else if (status.equals("close")) {
            led1.low();
            return "led close";
        }
        return "command error";
    }
}
```

一个简单的 Controller，将传过来的 Get 请求后的参数，作为控制 LED 灯亮灭的判断条件。

配置文件：

```properties
# 应用名称
spring.application.name=raspberry
# 应用服务 WEB 访问端口
server.port=9000
```

maven 打包好，上传到树莓派。

## 启动

将 led 灯珠（最好接个电阻，防止电流过大烧掉）正极接到 GPIO_01 上，负极接地。

打开腾讯云8090端口

```
firewall-cmd --zone=public --add-port=8090/tcp --permanent
firewall-cmd --reload
```

在腾讯云上，启动 frps

```
frps -c frps.ini
```

在树莓派上，启动 frpc

```
frpc -c frpc.ini
```

启动树莓派上的Springboot项目

```
java -jar raspberry-0.0.1-SNAPSHOT.jar
```

打开postman，发送一个get请求：

```
http://korilweb.cn:8090/command/open
http://korilweb.cn:8090/command/close
```

可以看到 LED 根据命令进行开关了。

---

## 简单的App

使用手机来控制 LED 是个不错的选择。

这里使用 Android Studio 开发一个简单的 App 应用。目前只有开关的接口，所以需要用到一个 Switch 组件来控制 LED 亮灭。

activity_main.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <Switch
        android:id="@+id/switch1"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="LED"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

在 MainActivity 中编写对应的逻辑代码

```java
package com.example.myapplication;

import androidx.appcompat.app.AppCompatActivity;

import android.annotation.SuppressLint;
import android.os.Bundle;
import android.widget.CompoundButton;
import android.widget.Switch;
import android.widget.TextView;

import java.io.IOException;

import okhttp3.OkHttp;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;

public class MainActivity extends AppCompatActivity {

    final static OkHttpClient client = new OkHttpClient();
    final static String url = "http://korilweb.cn:8090/command/";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        @SuppressLint("UseSwitchCompatOrMaterialCode") final Switch ledSwitch = findViewById(R.id.switch1);

        ledSwitch.setOnCheckedChangeListener((buttonView, isChecked) -> {
            if (isChecked) {
                System.out.println("开启");
                new Thread(() -> controlLed("open")).start();
            } else {
                System.out.println("未开启");
                new Thread(() -> controlLed("close")).start();
            }
        });
    }

    /**
     * 发送 get 请求，控制 led 小灯的亮灭
     * @param command 命令参数
     */
    protected void controlLed(String command) {
        Request request = new Request.Builder().url(url + command).build();
        try {
            Response response = client.newCall(request).execute();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

这里使用 okhttp 来发送请求，需要在 Gradle 中添加：

```groovy
implementation "com.squareup.okhttp3:okhttp:4.9.0"
```

最后打包成 apk 即可。

---

## 参考

https://shumeipai.nxez.com/2021/01/20/how-to-install-java-jdk-on-raspberry-pi.html

https://blog.csdn.net/qq_33259323/article/details/104441715?utm_medium=distribute.pc_relevant.none-task-blog-baidujs_title-0&spm=1001.2101.3001.4242

https://zhuanlan.zhihu.com/p/100355512
