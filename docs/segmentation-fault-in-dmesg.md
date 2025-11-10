---
date: 2021-04-12
tags: [debug]
---

# segfault in dmesg

偶尔看到自己或者客户的/var/log/message 日志出现segfault, 查询了一下相关信息
```
Apr  6 09:43:37 icm kernel: rhsm-icon[13402]: segfault at 12b0000 ip 0000003c89845c00 sp 00007ffce18396e0 error 4 in libglib-2.0.so.0.2800.8[3c89800000+115000]
``` 

## 解释

- address (after the at) - the location in memory the code is trying to access (it's likely that 10 and 11 are offsets from a pointer we expect to be set to a valid value but which is instead pointing to 0)
- ip - instruction pointer, ie. where the code which is trying to do this lives
- sp - stack pointer
- error - An error code for page faults; see below for what this means on x86.
```
/*
 * Page fault error code bits:
 *
 *   bit 0 ==    0: no page found       1: protection fault
 *   bit 1 ==    0: read access         1: write access
 *   bit 2 ==    0: kernel-mode access  1: user-mode access
 *   bit 3 ==                           1: use of reserved bit detected
 *   bit 4 ==                           1: fault was an instruction fetch
 */
```

## message日志

使用`dmesg`打印ring buffer的内容，关于硬件和i/o的信息

## coredump

```
 A core file is an image of a process that has crashed It contains all process information pertinent to debugging: contents of hardware registers, process status, and process data. Gdb will allow you use this file to determine where your program crashed. 
```

## 复现

```
void foo(){
    int *p = 0;
    *p = 100;
}

int main(){
  foo();
}
```

```
[ 5902.293905] a.out[6085]: segfault at 0 ip 000055c0eddca129 sp 00007ffe65372110 error 6 in a.out[55c0eddca000+1000]
[ 5902.293916] Code: 00 c3 66 66 2e 0f 1f 84 00 00 00 00 00 0f 1f 40 00 f3 0f 1e fa e9 67 ff ff ff 55 48 89 e5 48 c7 45 f8 00 00 00 00 48 8b 45 f8 <c7> 00 64 00 00 00 90 5d c3 55 48 89 e5 b8 00 00 00 00 e8 d9 ff ff
```

```
(gdb) info registers 
rax            0x0                 0
rbx            0x55c0eddca150      94287112741200
rcx            0x7faa085eb598      140368261592472
rdx            0x7ffe65372228      140730596532776
rsi            0x7ffe65372218      140730596532760
rdi            0x1                 1
rbp            0x7ffe65372110      0x7ffe65372110
rsp            0x7ffe65372110      0x7ffe65372110
r8             0x0                 0
r9             0x7faa08621070      140368261812336
r10            0x69682ac           110527148
r11            0x202               514
r12            0x55c0eddca020      94287112740896
r13            0x0                 0
r14            0x0                 0
r15            0x0                 0
rip            0x55c0eddca129      0x55c0eddca129 <foo+16>
eflags         0x10246             [ PF ZF IF RF ]
cs             0x33                51
ss             0x2b                43
ds             0x0                 0
es             0x0                 0
fs             0x0                 0
gs             0x0                 0
```

## addr2line

```
addr2line -e yourSegfaultingProgram 00007f9bebcca90d
```
