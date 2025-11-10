---
comments: true
date: 2025-07-21
tags: [GPIO,esp32]
---

# Learn GPIO
GPIO是通用目的的输入输出引脚，可以配置很多功能，因为每个GPIO引脚都有许多寄存器，例如4个控制寄存器，2个数据寄存器，1个复位寄存器，1个时钟寄存器，2个复用功能寄存器。  
树莓派，ESP32, arduino（Atmel328p）都有GPIO引脚。

## GPIO 的几种模式

* 输出： 点亮灯泡。又分为推挽（PUSH_PULL） 这是默认的输出模式（高/低电平）； 开漏（OPEN_DRAIN）用得少，这是只能将引脚设置成低电平，要靠外部电路实现高电平，应用在例如i2c总线。
* 输入： 按钮控制。又分为内部上拉 PULL_UP 引脚默认高电平； 内部下拉 PULL_DOWN 引脚默认低电平。

## GPIO 输出数字信号和PWM
无论是PUSH_PULL还是OPEN_DRAIN，GPIO引脚只有高电平和低电平两种状态，这也叫数字信号，例如下面的发送摩斯码。 PWM信号实际是高低电平的串行信号组（例如__---___---___），ESP32的所有GPIO支持输出PWM信号，但推荐使用 GPIO 4, 5, 12–14, 18–19, 21–23, 25–27, 32–33。 PWD信号用来驱动步进电机。

## GPIO 输出模拟信号
模拟信号（analog）可以驱动喇叭播放声音，ESP32中只有GPIO 25-26 支持输出模拟信号。
```py
from machine import DAC, Pin

dac = DAC(Pin(25))     # Use GPIO25
dac.write(128)         # Output ~1.65V (128/255 × 3.3V)
```

## GPIO 按钮实现发送摩斯码

```py
from machine import Pin
import time

button = Pin(15, Pin.IN, Pin.PULL_UP)
morse = ""
press_start = 0
last_event_time = time.ticks_ms()

MORSE_TABLE = {
    ".-":"A","-...":"B","-.-.":"C","-..":"D",".":"E",
    "..-.":"F","--.":"G","....":"H","..":"I",".---":"J",
    "-.-":"K",".-..":"L","--":"M","-.":"N","---":"O",
    ".--.":"P","--.-":"Q",".-.":"R","...":"S","-":"T",
    "..-":"U","...-":"V",".--":"W","-..-":"X","-.--":"Y",
    "--..":"Z", "": " "
}

def decode(morse_code):
    return MORSE_TABLE.get(morse_code, "?")

while True:
    now = time.ticks_ms()

    if button.value() == 0:  # Press detected
        press_start = now
        while button.value() == 0:
            time.sleep(0.01)
        press_duration = time.ticks_diff(time.ticks_ms(), press_start)
        symbol = "." if press_duration < 300 else "-"
        morse += symbol
        last_event_time = time.ticks_ms()
        print("Symbol:", symbol)

    # Check for end of character
    if morse and time.ticks_diff(now, last_event_time) > 1000:
        char = decode(morse)
        print("Char:", char)
        morse = ""

    time.sleep(0.01)

```

## 引用
https://blog.51cto.com/u_4029519/13530340
https://blog.csdn.net/makeryzx/article/details/78915955