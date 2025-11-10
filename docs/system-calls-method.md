---
date: 2020-11-09
tags: [linux,api]
---

# system calls method

之前学汇编发现教材和实际的有出入， 书上写的int, 但是汇编不通过，而gcc 反汇编的结果是调用syscall。  
原来这是两种方式调用方式即： int  0x80 和 syscall  

除此之外还有一个名词是vdso,  很多elf文件会链接这个vdso库

```
ldd a.out                                                                                                                                                                                                                   √ 19:03:30 
	linux-vdso.so.1 (0x00007fffb3de0000)
	libc.so.6 => /usr/lib/libc.so.6 (0x00007f5b48d4a000)
	/lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007f5b48f36000)

```

## 词汇说明

`int 0x80` 即80中断， 是最老的系统函数调用方式
`syscall/sysret` 是amd64 制定的标准， 也是目前的x86 64位的标准，即`amd64`
`sysenter/syssysexit` 是inter制定的x86 64位标准， 目前已被放弃
`vdso` 是linux内核虚拟出的so, 实现了int 80 和 syscall，调用方式为 `vsyscall`

## 系统函数调用路径 

系统调用多被封装成库函数提供给应用程序调用，应用程序调用库函数后，由 glibc 库负责进入内核调用系统调用函数。
即`用户函数-> glibc -> 系统调用`

## int 0x80

int 即是interrupt 中断， 0x80是IDT上注册的中断向量， 每个编号对应一个处理函数handle， linux的0x80的handle即是内核，即系统调用。
所以不同的系统设置的0x80的handle可能不同

调用方式：首先是将参数复制到寄存器， 参数包括系统调用编号和传入参数，然后执行 init  0x80 
例如，以下的进程退出的系统调用
```
.data
    s:
        .ascii "hello world\n"
        len = . - s
.text
    .global _start
    _start:

        movl $4, %eax   /* write system call number */
        movl $1, %ebx   /* stdout */
        movl $s, %ecx   /* the data to print */
        movl $len, %edx /* length of the buffer */
        int $0x80

        movl $1, %eax   /* 退出的系统调用编号 */
        movl $0, %ebx   /* exit status */
        int $0x80
```

## vdos 

vdos即 linux-vdso.so.1， 几乎很多elf 都会链接这个库，但其实他并不是真实存在的so文件，  
而是由内核虚拟的文件，再映射到用户的进程来调用。

vdos  是对以下几个函数的实现，称作快速调用

```
#define __NR_gettimeofday 96 //0x60
#define __NR_time 201 //0xc9
#define __NR_clock_gettime 228 //0xE4
#define __NR_getcpu 309 //0x135
```

## 对比

所以以上总结其实就3种方式， int ，syscall/sysret ， vdso

int 0x80 方式很慢，所以出现了syscall 即快速调用

执行区别
```
在 x86 保护模式中，处理 INT 中断指令时，CPU 首先从中断描述表 IDT 取出对应的门描述符，判断门描述符的种类，然后检查门描述符的级别 DPL 和 INT 指令调用者的级别 CPL，当 CPL<=DPL 也就是说 INT 调用者级别高于描述符指定级别时，才能成功调用，最后再根据描述符的内容，进行压栈、跳转、权限级别提升。内核代码执行完毕之后，调用 IRET 指令返回，IRET 指令恢复用户栈，并跳转会低级别的代码。

其实，在发生系统调用，由 Ring3 进入 Ring0 的这个过程浪费了不少的 CPU 周期，例如，系统调用必然需要由 Ring3 进入 Ring0（由内核调用 INT 指令的方式除外，这多半属于 Hacker 的内核模块所为），权限提升之前和之后的级别是固定的，CPL 肯定是 3，而 INT 80 的 DPL 肯定也是 3，这样 CPU 检查门描述符的 DPL 和调用者的 CPL 就是完全没必要。正是由于如此，Intel x86 CPU 从 PII 300（Family 6，Model 3，Stepping 3）之后，开始支持新的系统调用指令 sysenter/sysexit。sysenter 指令用于由 Ring3 进入 Ring0，SYSEXIT 指令用于由 Ring0 返回 Ring3。由于没有特权级别检查的处理，也没有压栈的操作，所以执行速度比 INT n/IRET 快了不少。
```

返回的区别
```
在 Intel 的手册中，还提到了 sysenter/sysexit 和 int n/iret 指令的一个区别，那就是 sysenter/sysexit 指令并不成对，sysenter 指令并不会把 SYSEXIT 所需的返回地址压栈，sysexit 返回的地址并不一定是 sysenter 指令的下一个指令地址。调用 sysenter/sysexit 指令地址的跳转是通过设置一组特殊寄存器实现的。
```

vdos的局限（syscall）
```
而"快速系统调用指令"比起中断方式的系统调用方式，还存在一定局限，例如无法在一个系统调用处理过程中再通过"快速系统调用指令"调用别的系统调用。因此，并不一定每个系统调用都需要通过"快速系统调用指令"来实现。比如，对于复杂的系统调用例如 fork，两种系统调用方式的时间差和系统调用本身运行消耗的时间来比，可以忽略不计，此处采取"快速系统调用指令"方式没有什么必要。而真正应该使用"快速系统调用指令"方式的，是那些本身运行时间很短，对时间精确性要求高的系统调用，例如 getuid、gettimeofday 等等。
```

## 最后总结

int 是最老的方式，目前用amd64的 syscall 方式， 而vdso是基于syscall实现的快速调用。  
只有在调用clock_gettime、gettimeofday、getcpu、time这些系统调用时，才会使用vdso，其他系统调用是通过syscall实现的


