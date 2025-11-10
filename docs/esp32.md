---
date: 2024-12-06
tags: [esp,arduino]
---

# 玩转ESP32

ESP32的生态比较好，类似arduino, 但是集成了wifi和蓝牙，有的甚至集成了gps，功能很强大而且便宜。最近淘宝买了一个ESP32-USB-Geek， 带一个LCD屏幕和几个扩展接口，还有闪存口可以读写。

  
![[esp32-usb-geek.png.png]]


## 编译烧录

ESP32 有3种方式： 官方的idf， mpy, arduino。

### 官方工具

官方的idf分vscode插件和python脚本， vscode中没有成功，不过idf.py可以编译和烧录。

  

根据官方文档写的下载安装idf，解压后执行脚本。安装脚本依赖python-venv， 而我最常用vituralenv

```sh

bash v5.3.1/esp-idf/install.sh

```

  

进入淘宝店例子, 用cmake管理的。 在跟目录执行

```sh

idf.py set-target esp32-s3

```

这种方式支持非常复杂的配置

```sh

idf.py menuconfig

```

  

编译刷入

```

idf.py build

idf.py -p /dev/ttyACM0 flash

```

  

官方的方式最能了解物理设备了。看日志发现他生成了bin文件，并从0位置刷入bin文件。

  

### mpy thonny ide

最好用pip安装最新的。 将python上传到esp后，点运行就可以了，还能在REPL中仿真调试，非常方便。

  

### arduino

arduino ide gui版本编译失败，而且因为是Linux版是用java写的，操作非常慢。改用arduio-cli脚本成功， 官方文档也说了arduino-cli非常强大。这里说脚本使用方法。

  

#### 初始化配置

1. 首先下载压缩包，解压到PATH目录下。

2. 初始化配置， `arduino-cli config init` 可以生成配置文件'~/.arduino15/arduino-cli.yaml'。

3. 修改配置文件， 根据淘宝老板说的，ESP32的板子要从另外第三方源中， 因为这个源是githubcontent需要梯子才能访问，修改成如下

  

```text

$ cat ~/.arduino15/arduino-cli.yaml

board_manager:

additional_urls: ['https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json']

  

network:

proxy: 'socks5://ip:port'

```

  

#### 安装板子库和依赖库

当没有下载板子时，查看连接的设备会显示Unknown

```

$ arduino-cli board list

Port Protocol Type Board Name FQBN Core

/dev/ttyACM0 serial Serial Port (USB) Unknown

```

  

显示搜索所有板子（包括在添加的第三方源里的）, 这里搜索ESP32核心板， 淘宝老板说，我的开发板叫 ESP32S3 Dev Module

```

$ arduino-cli board listall ESP32S3 Dev Module

ESP32S3 Dev Module esp32:esp32:esp32s3

```

  

安装板子module， 比较大要等待一会。 tips： 板子的包叫module, 软件依赖库叫library

```

$ arduino-cli core install esp32:esp32

```

  

搜索已安装的安装核心板

```

$ arduino-cli core search esp32

ID Version Name

arduino:esp32 2.0.18-arduino.5 Arduino ESP32 Boards

esp32:esp32 3.0.7 esp32

```

  

#### 编译

编译淘宝店的例子LCD_Button，要在LCD_Button外面执行

  

arduino-cli compile --fqbn esp32:esp32:esp32s3 LCD_Button

  

#### 上传到板子

arduino-cli upload -p /dev/ttyACM0 --fqbn esp32:esp32:esp32s3 LCD_Button

有时候失败了，要按住Boot键插入电脑再松开，就能正常上传。

```

esptool.py v4.6

Serial port /dev/ttyACM0

Connecting...

Traceback (most recent call last):

...

OSError: [Errno 71] Protocol error

Failed uploading: uploading error: exit status 1

```

看这个日志，原来arduino也是调用的官方的idf工具的esptool脚本。 重新插入USB， 再次查看已连接的开发板就不会显示Unknown了

  

```sh

$ arduino-cli board list

Port Protocol Type Board Name FQBN Core

/dev/ttyACM0 serial Serial Port (USB) ESP32 Family Device esp32:esp32:esp32_family esp32:esp32

```

  

## 连接串口UART

烧录了淘宝店给的串口收发例子，代码非常简单， 然后用我的USB转ttl工具测试成功。

```py

import machine

  

uart = machine.UART(1, baudrate=115200, tx=machine.Pin(43), rx=machine.Pin(44))

  

def send_data(data):

uart.write(data)

  

def receive_data():

if uart.any():

data = uart.read()

print("Received data:",data)

  
  

while True:

send_data("Hello UART")

receive_data()

```

  

然而这里是不对的， 串口转TTL和ESP的USB都要连接到电脑上。单连一个都是进了mpy的repl。

![[usb-ttl-esp32-uart.png.png]]