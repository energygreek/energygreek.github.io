---
date: 2022-04-10
tags: [51单片机]
---

# 在Linux下使用sdcc进行单片机编译
周末研究了下单片机，本来准备写个定时器来控制电扇开关，发现比起树梅派，单片机实在不太方便

## 开发环境搭建

Windows大家都用Keil, 一站式比较方便，但这是个收费软件。而在Linux下，使用到一些组合来实现编译和烧录

1. sdcc 开源的编译器，类似gcc
2. [stcflash](https://github.com/laborer/stcflash) github上的开发者开发的烧录脚本

sdcc 在Debian有包可以直接安装
```
# apt install sdcc
```
再 clone github的代码到本地，需要使用root权限执行烧录。而我执行是发现缺少python serial包，要全局安装就使用apt命令

```
# apt install python3-serial
```

这样环境就搭建好了

## 编写代码

用c语言开发时要引入比较的定义头文件，我的单片机是好多年前买的国产`stc 89C52RC`， 是基于51的拓展版本，于是用了sdcc的`8052.h`头，路径在
```c
// /usr/share/sdcc/include/mcs51/8052.h
#ifndef REG8052_H
#define REG8052_H
  
#include <8051.h>     /* load definitions for the 8051 core */
  
#ifdef REG8051_H
#undef REG8051_H
#endif
```

不过不需要我们指定路径，sdcc会自动引入。窥视其内容可以发现它包含了8051的东西，所以说52是基于51的


## 回顾一下51单片机知识

1. 单片机的寄存器初始是0, 而引脚初始都是高点位
2. 定时器和计数器的触发事件都是靠中断，中断执行当前（main）中的函数而执行中断响应函数，这跟Unix内核是一样的。 单片机还具有多个优先级的中断，可以嵌套
3. 中断的触发靠计数寄存器（TLx和THx）溢出，这两个都是8位寄存器，所以同时使用时，在2^16时溢出。但也有只用其中一个寄存器的时候，这由`TMOD`寄存器控制

### TMOD寄存器

TMOD是不可按位访问的寄存器，意思是只能一次性赋值所有位。 TMOD控制着两个定时器/计数器0和1

![[timer.png]]

* TMOD高四位用于设置定时器/计数器1，低四位用于设置定时器/计数器0；
* GATE寄存器决定是否使用外部INTx控制，即软件自动启动，还是外部启动，这里赋值0
  * GATE=1时，“与门”的输出信号K由INTx输入电平和TRx位的状态一起决定(即此时K=TRx·INTx)，当且仅当TRx=1,INTx=1(高电平)时，计数启动；否则，计数停止。当INT0引脚为高平时且TR0置位，TR0=1；启动定时器T0
  * GATE=0时，“A”输出恒为1，“B”的值由TRx决定，当TR0=1,启动定时器/计数器T0，当TR1=1，启动定时器/计数器T1
* CT位为1时选择计数器模式，为0时选择定时器模式
* M1,M0用于选择工作方式，当M0=1,M1=0时，使用TL0和TH0的16位来计数

### TCON寄存器

![[TCON.png]]

### 定时器初始化

这里使用定时器0
```
void Timer0Init(void) // 1毫秒@12.000MHz
{
    TMOD = 0x01; //设置定时器模式 0000 0001
    // TCON
    TF0 = 0; //初始化溢出标志
    TR0 = 1; //定时器0开始计时
    // 计数器初始值，再触发一次就溢出了
    TH0 = 0xFF; //设置定时初值
    TL0 = 0xFE; //设置定时初值
    // 中断通路
    EA = 1; 
    ET0 = 1;
    PT0 = 0;
}
```

### 计数器模式

这里使用计数器0
```c
void Count0Init(void) // 1毫秒@12.000MHz
{ 
    TMOD = 0x05; //设置计数器模式 0000 0101

    // TCON
    TF0 = 0; 
    TR0 = 1; //定时器0开始计时

    // 计数器初始值，再触发一次就溢出了
    TH0 = 0xFF;  //设置定时初值
    TL0 = 0xFE;  //设置定时初值

    // 中断通路
    EA = 1;
    ET0 = 1;
    PT0 = 0;
}
```

### 中断响应函数

sdcc和keil有一些区别，中断标志符使用`__interrupt`
```c
static unsigned int T0Count = 0;
void timer0_routine() __interrupt (1) {
  T0Count++;
  TH0 = 0xFF; //设置定时初值
  TL0 = 0xFE; //设置定时初值
  if (T0Count == 3) {
    T0Count = 0;
    P1_1 = 0;  // 0输出低电平，灯泡亮
  }
}
```

### 完整代码

```c
// main.c
#include "8052.h"

void Timer0Init(void) // 1毫秒@12.000MHz
{
  TMOD = 0x01;
  TF0 = 0;
  TR0 = 1;
  TH0 = 0xFF;
  TL0 = 0xFE;
  EA = 1;
  ET0 = 1;
  PT0 = 0;
}

void main(void) {

  Timer1Init();
  while (1)
    ;
}

unsigned int T0Count = 0;
void timer0_routine() __interrupt(1) {
  T0Count++;
  TH0 = 0xFF;
  TL0 = 0xFE;
  if (T0Count == 3) {
    T0Count = 0;
    P1_1 = 0;
  }
}
```

## 编译和烧录

代码保存为main.c

```
# 使用sdcc编译
sdcc main.c
将sdcc生成的ihx文件转为hex文件
packihx main.ihx  > main.hex   

```

烧录到单片机
```bash
sudo python ~/github/stcflash/stcflash.py main.hex
Connect to /dev/ttyUSB0 at baudrate 2400
Detecting target... done
 FOSC: 11.955MHz
 Model: STC89C52RC (ver4.3C) 
 ROM: 8KB
 [X] Reset stops watchdog
 [X] Internal XRAM
 [X] Normal ALE pin
 [X] Full gain oscillator
 [X] Not erase data EEPROM
 [X] Download regardless of P1
 [X] 12T mode
Baudrate: 38400
Erasing target... done
Size of the binary: 147
Programming: #################### done
Setting options... done
```
