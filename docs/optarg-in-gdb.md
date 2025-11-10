---
date: 2022-08-19
tags: [gdb]
---

# gdb中打印optarg永远是0x0, 但是依然能传递参数


## 发现两个optarg不相同

```
(gdb) p &optarg
$7 = (char **) 0x7ffff7fa5840 <optarg>
(gdb) x/1gx &optarg
0x7ffff7fa5840 <optarg>:        0x0000000000000000
(gdb) x/1gx $rip+7+0x2edf    // from disassembly of what *actually* gets passed to printf
0x555555558040 <optarg@GLIBC_2.2.5>:    0x00007fffffffdf32
```
