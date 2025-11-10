---
date: 2020-01-10
tags: [c, cpp, gcc]
---

# gcc 会优化很多内存相关的调用

他们通常在'string.h'里，比如memcpy, 当编译器通过上下文能推断出传入参数类型，libc的mempcy就会替换为gcc的memcpy, 这可能导致问题

## 举例

知乎评论的，release版本的memset被替换, 导致跟debug版本不同行为bug
```
https://www.zhihu.com/question/435258463/answer/1969561682

```

## 内存地址对齐

如果内存地址对齐的，那么'取值地址被取值长度求模为0'，例如`int i`, 则要求`(void*)&i % sizeof(int) == 0`  

## 原因

当同时定义多个变量时，不考虑编译器优化，则他们在内存的位置相邻。按照不同的对齐长度，有不同的占用大小  
这里以结构体举例

```
struct bar{
	int interger;
        char charactor;
        long long longlong;
};
```
使用`#pragma pack(4)` 设置按4字节对齐，默认也是4字节，那么`sizeof(struct abc) == 16`
使用gdb打印内存地址，可见地址都对齐的: '0x7fffffffe430&4==0' '0x7fffffffe438%8==0'
```
(gdb) p sizeof(struct abc)
$5 = 16
(gdb) p &a.interger 
$1 = (int *) 0x7fffffffe430
(gdb) p &a.charactor 
$2 = 0x7fffffffe434 "\232\177"
(gdb) p &a.longlong 
$3 = (long long *) 0x7fffffffe438
(gdb) p &a
$4 = (abc *) 0x7fffffffe430
```

顺便提一下，指针指向的是内存段的低地址，所以有个面试题是判断系统是大端小端的方法如下  
通过如下强转int为char, 如果`c==0x12`则为大端，否则为小端系统
```
int i=0x12345678;
char c=(char)i;
```

使用`#pragma pack(1)` 设置为按1字节对齐，则上面的结构体大小为'13'，显然更省内存了  
但通过取地址会发现地址未对齐, 求模不为0。 但我自己使用gcc测试地址依旧对齐的，原因是编译器会进行自然 对齐

## 自然对齐

上面的`#pragma pack(1)`虽然使得longlong的地址对于bar来说偏移了5字节，但编译器始终能调整bar的起始地址使得longlong的地址对齐
```
595fac88 
c3c5a2d8
105d3ab8
```

## 不对齐的坏处

虽然上面说了编译器能自然对齐，但还是会真的出现不对齐访问。由于对内存的访问涉及到具体硬件的实现，所以一般会有几种影响
1. 硬件平台支持非对齐访问，但性能有损耗
2. 抛出异常给cpu,由cpu来校准，性能有更大损耗
3. 平台不支持，但也不抛异常，获取的是错误的值，导致难定位的异常

## 引用

https://www.kernel.org/doc/html/latest/core-api/unaligned-memory-access.html
