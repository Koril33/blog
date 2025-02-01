---
title: "NodeMCU发送数据给树莓派"
date: 2021-06-18T11:20:10+08:00
summary: "MicroPython+NodeMCU上传数据到树莓派"
featured_image: "images/background.jpg"
toc: true
---

## NodeMCU

NodeMCU 是一个开源的物联网平台。 该平台基于eLua开源项目，底层使用ESP8266 sdk 0.9.5版本。 NodeMCU包含了可以运行在esp8266Wi-FiSoC芯片之上的固件,以及基于ESP-12模组的硬件。用 C 实现了 Lua、Python 的编程环境，可以很方便的实现一些物联网应用。

---

## 烧录固件

我对于 Lua 语言不太熟悉，所以这里采用 MicroPython 的方案，注意检查连接 NodeMCU 的数据线，如果只是充电线，电脑是无法识别的。插上后，如果没有反应，可以手动安装 CH340 驱动，这是一个 USB 转串口的驱动。最后，从设备管理器中，可以看到连接的是哪个串口。

### 下载固件

Windows 需要有 Python 环境，这里不再赘述如何安装 Python。

固件下载地址：http://www.micropython.org/download/esp8266/

下载：esp8266-20210418-v1.15.bin (elf, map) (latest)

### 烧录固件

这里采用 esptool 的脚本来进行烧录，esptool 通过 pip 来下载

```
python -m pip install esptool
```

通过“设备管理器”确认NodeMCU开发板的连接端口号，然后清除开发板固件信息

```
python -m esptool --port COM5 erase_flash
```

最后，刷入 MicroPython 固件

```
python -m esptool --port COM5 --baud 115200 write_flash --flash_size=detect 0 D:\InstallPackage\esp8266-20210418-v1.15.bin
```

### 串口工具连接NodeMCU

可以选择的工具很多，XShell、putty、Windows10的串口调试助手。

参数调整一下，都可以连接成功，选择正确的端口（我这里是COM5），波特率选择115200，其他不用动。点击打开串口，可以出现 python 脚本的界面。

---

## 点灯

需要注意的是，NodeMCU Dev kit引脚的编号与Nodemcu的内部GPIO编号不是一个编号。我之前以为板子上的D0对应0，D1对应1，结果怎么都不对。

这里给出映射关系：

| NodeMCU开发板Pin | ESP8266 内部 GPIO Pin 编号 |
| ---------------- | -------------------------- |
| D0               | GPIO16                     |
| D1               | GPIO5                      |
| D2               | GPIO4                      |
| D3               | GPIO0                      |
| D4               | GPIO2                      |
| D5               | GPIO14                     |
| D6               | GPIO12                     |
| D7               | GPIO13                     |
| D8               | GPIO15                     |
| D9/RX            | GPIO3                      |
| D10/TX           | GPIO1                      |
| D11/SD2          | GPIO9                      |
| D12/SD3          | GPIO10                     |

把 LED 正极连接上 D3，负极接地，然后通过命令行点亮 LED，根据上面的表格可以知道，D3 对应 GPIO0

```python
>>> from machine import Pin
>>> led = Pin(0, Pin.OUT)
>>> led.on()
>>> led.off()
```

on 就是点亮，off 就是熄灭

---

## 联网

通过内置的 network 库，可以连接上 WiFi

```python
>>> import network
>>> wlan.connect("djhxiaomi", "0987654321")
>>> wlan.isconnected()
True
```

---

## DHT11

DHT11 DHT11是一款有已校准数字信号输出的温湿度传感器。 其精度湿度±5%RH， 温度±2℃，量程湿度5-95%RH， 温度0~+50℃。三个引脚，VCC 接 NodeMCU的3v，GND 接地，信号线接某个引脚，这里我接的是 D2，对应 GPIO4。

```python
>>> import dht
>>> dh = dht.DHT11(machine.Pin(4))
>>> dh.measure()
>>> dh.temperature()
30
>>> dh.humidity()
56
```

---

## 发送请求

通过内置的 urequests 库，可以发送 HTTP 请求，这里简单说下 GET 和 POST

### GET

```python
>>> import urequests
>>> response = urequests.get("http://api.com/test")
>>> print(response.text)
```

### POST

```python
>>> import urequests
>>> import json
>>> api_url = "http://api.com/test"
>>> sensor_data = {"name" : "koril", "status" : 100}
>>> headers = {"Content-type" : "application/json"}
>>> response = urequests.post(api_url, data = json.dumps(sensor_data), headers = headers)
>>> print(response.text)
```

---

## 上传代码到NodeMCU

在命令行下一行行编写很麻烦，所以我们可以直接写好代码，上传到 NodeMCU 里，这里用的工具是 MicroPython File Uploader，一个小巧的 GUI 软件

下载地址：https://www.wbudowane.pl/download/

编写好 main.py，文件名一定得是 main.py

```python
import time
from machine import Pin

led = Pin(0, Pin.OUT)
def blink():
    while True:
        led.on()
        time.sleep(1)
        led.off()
        time.sleep(1)
blink()
```

然后直接通过软件，打开串口，选中 main.py 文件，直接 send 即可。

---

## 将DHT11数据发送到树莓派上

首先 NodeMCU 和树莓派都需要联网，将树莓派当作服务器，接收来自 NodeMCU 的数据，我将树莓派的端口映射到了公网 IP 上，这样 NodeMCU 就可以直接向公网 IP 发送请求，如果没有进行内网穿透，也可以将 NodeMCU 和树莓派置于同一个局域网内，通过局域网来收发数据。

NodeMCU 的 main.py

```python
import network
import urequests
import time
import machine
import dht
import json

dh = dht.DHT11(machine.Pin(4))

# 树莓派上的服务端口
api_url = "http://korilweb.cn:8090/data"

# 因为有时候带到不同的地方，学校，家里，还有公司
# 所以用一个 dict 存放所有的 WiFi 信息
wifi_dict = {
    "home" : ["TP-LINK_832B", "sqf1226lhw"],
    "phone" : ["djhxiaomi", "0987654321"]
}

# 对某个WiFi进行连接
def do_connect(SSID, PASSWORD):
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    if not wlan.isconnected():
        print("connecting to " + SSID)
        wlan.connect(SSID, PASSWORD)
        
    start = time.time()
    while not wlan.isconnected():
        time.sleep(1)
        # 超过5秒，认为连接超时
        if time.time() - start > 5:
            print("connect timeout!")
            break
    # 连接成功，返回 True，打印 wlan 信息
    if wlan.isconnected():
        print('network config:', wlan.ifconfig())
        return True
    # 连接失败，返回 False
    return False

# 对 WiFi 字典进行遍历，某一个连接成功，就发送数据
def connect_wifi(wifi_dict):
    for wifi_info in wifi_dict:
        is_connected = do_connect(wifi_dict[wifi_info][0], wifi_dict[wifi_info][1])
        if is_connected:
            time.sleep(1)
            while True:
                send_data()
                time.sleep(1)
    print("no wifi")

# 发送传感器数据
def send_data():
    tempAndHumidity = getTempAndHumidity()
    sensor_data = {
        "temperature" : tempAndHumidity[0],
        "humidity" : tempAndHumidity[1]
    }
    headers = {
        "Content-Type" : "application/json"
    }

    response = urequests.post(
        api_url,
        data = json.dumps(sensor_data), 
        headers = headers
    )
    print(response.text)

# 获取 DHT11 的数据，温度和湿度，以元组的形式返回
def getTempAndHumidity():
    dh.measure()
    return (dh.temperature(), dh.humidity())


connect_wifi(wifi_dict)
```

树莓派服务端代码

```java
package com.example.raspberry.controller;

import com.example.raspberry.entity.Sensor;
import org.springframework.web.bind.annotation.*;

@RestController
public class HelloController {

    @PostMapping(value = "/data")
    public String data(@RequestBody Sensor sensor) {
        System.out.println("temp: " + sensor.getTemperature());
        System.out.println("humidity: " + sensor.getHumidity());
        return "get data";
    }
}
```

Sensor 是 DHT11 的类

```java
package com.example.raspberry.entity;

public class Sensor {

    // 温度
    private Integer temperature;
    // 湿度
    private Integer humidity;

    public Integer getTemperature() {
        return temperature;
    }

    public void setTemperature(Integer temperature) {
        this.temperature = temperature;
    }

    public Integer getHumidity() {
        return humidity;
    }

    public void setHumidity(Integer humidity) {
        this.humidity = humidity;
    }

    @Override
    public String toString() {
        return "Sensor{" +
                "temperature=" + temperature +
                ", humidity=" + humidity +
                '}';
    }
}

```

---

## 运行

树莓派上：

```
java -jar raspberry-0.0.1-SNAPSHOT.jar
```

然后将 main.py 上传到NodeMCU，可以看到树莓派打印的日志信息，显示了传感器的温湿度。

---

## 参考

https://blog.csdn.net/niupipiniupipi/article/details/80956404

http://www.taichi-maker.com/homepage/esp8266-nodemcu-iot/iot-micropython/micropython-nodemcu-firmware/

https://blog.csdn.net/wzy15965343032/article/details/101768710

http://docs.micropython.org/en/latest/esp8266/quickref.html#

https://www.wbudowane.pl/

https://www.basemu.com/nodemcu-gpio-interface.html
