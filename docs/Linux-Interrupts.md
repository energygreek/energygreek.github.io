---
date: 2020-10-19
tags: [Linux,内核原理]
---

# Linux 中断请求

中断请求的英文是IRQ（Interrupt Request）,是用来驱动CPU正常工作的重要机制。 中断根据源头分类成：
由外设发出的硬件中断，软件发出的软中断，以及异常组成。

以前，每个外设都连接一根线到PIC(programmable Interrupt circuit)芯片，有PIC芯片来发生数据到CPU,并将CPU的INTR引脚置位。
现在，CPU集成了PIC成为APIC(Advanced PIC)。

## 中断分类
中断分为3类型

### 硬件中断
鼠标和键盘，还有IO设备都会发出硬件中断，例如网卡的数据到了，就会出发中断请求（IRQ），
这个请求会触发CPU去执行ISR, 而这个ISR需要在系统启动的时候注册的。一个Vector表。

## 软件中断
当播放电影的时候，声音和画面的同步非常重要，这是由系统的定时器[jiffies](https://elinux.org/Kernel_Timer_Systems)来不停地调度声音播放器来准确播放对应的声音。
软中断在实时操作系统的作用也非常重要。

### 异常中断

异常中断又分为3类： 缺页异常(Faults)，陷入异常(Traps)，异常退出(Aborts)

例如当程序的内存被swap到硬盘了或者一段动态链接库还没有载入到内存里来，在程序走到这个位置之前，
程序就会提前发出缺页异常，当系统检查了权限和地址有效后（地址有效表示在逻辑地址范围内），
CPU开始为程序执行加载需要的数据，并清除这个异常。 所以这个异常是可以修正的，这个请求必须提前发出。

当GDB调试的时，设置断点会插入一条特殊的指令到程序里，当执行到断点位置就会出发Traps请求，软件的主导权由CPU交给GDB。

当程序触发一个异常例如0/0时，会出发退出异常，由cpu来清理堆栈。


## 请求的可阻塞性

对于硬件中断请求，CPU的决策是立即执行，对于软中断和异常，CPU采用尽量延后的策略。

## 查看系统的中断 

在中断执行 `watch -n1 "cat /proc/interrupts"`可以定时更新中断的次数

列的含义从左至右以此为：
```
 IRQ vector, interrupt count per CPU (0 .. n), the hardware source, the hardware source's channel information, and the name of the device that caused the IRQ.
```


