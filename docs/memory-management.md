---
date: 2021-12-01
tags: [memory, malloc] 
---

# 浅谈内存管理

这里记录一下看书所得，应用程序的内存管理，即glibc的malloc和相关函数

## 内存管理的重要性

过去Linux通过默认的dlmalloc来申请内存，但不支持多线程，这导致的竞争和性能降低，而ptmalloc2支持多线程，也成为了Linux新的默认方式  
ptmalloc2通过为每个线程单独维护一个帧，这样每个线程的申请和释放可以并行执行，但当线程过多时，线程也将共享帧。这种帧叫做`per thread arena`

## 主线程和线程的内存申请方式

主线程中使用brk来申请大块内存，其实`brk`不过是修改了寄存器的值。而且返回的大小通常要比malloc申请的多，如同测试中的在主线程中申请100个字节大小的内存，返回了132 KB。
```
55f7b402c000-55f7b404d000 rw-p 00000000 00:00 0                          [heap]
```
而在线程中，malloc使用mmap来申请内存，返回大小也比申请大小多，测试中的在线程中申请100字节，返回了64M的地址空间， 但只有8MB是可用的堆内存。
```
7fb738000000-7fb738021000 rw-p 00000000 00:00 0      
7fb738021000-7fb73c000000 ---p 00000000 00:00 0      
```

主线程中申请的地址大于Data Section的地址，说明使用的`brk`，而后者申请时用了mmap，对比附录文档可以看出32位和64位区别比较大

## arena

主线程和线程使用不同的arena，这样主线程和线程同时申请内存时不存在竞争。但不是所有的线程都独享一份arena, arena的个数限制如下

```
For 32 bit systems:
     Number of arena = 2 * number of cores.
For 64 bit systems:
     Number of arena = 8 * number of cores.
```

当线程个数太多时，会公用主线程的arena

### main arena 和 per thread arena

main arena 指在主线程（main）中使用的，而后者是子线程中使用，两者有很大区别  
主线程只有一块heap，所以没有heapinfo结构，而子线程有。主线程的`Arena header`是全局变量，所以存在于libc.so的Data Segment中  

线程的arena含有多个heap, 每个heap都有heapinfo结构，多个heapinfo组成链表。线程的多个heap共享一个`malloc_state` 也称为`Arena header`  

`Arena header`包含了bin、top chunk等信息  

每个heap有多个chunk，每个chunk有个`malloc_chunk`结构，其管理chunk的状态和其在heap中的偏移大小，有以下几种状态

- Allocated chunk 
- Free chunk
- Top chunk
- Last Remainder chunk

Top chunk 处于arena的顶部，在freelist没有合适大小的chunk时使用，如果Top chunk比申请的大，则又分为两部分（新Top chunk和用户申请的chunk）  

Last Remainder chunk, 是当用户请求small chunk时，会先遍历small bin和Unsorted bin， 如果都无法满足，则遍历其他bin。  
如果找到的chunk还有剩余，则这个chunk变为`Last Remainder chunk`，且将被加入到`Unsorted bin`  
这样的好处是多次申请small chunk时，会先向`Unsorted bin`的`Last Remainder chunk`申请， 所以小内存块会集中分布在`Last Remainder chunk`，提高小块内存的局部性  

## bin

当malloc的内存被free后（仅仅是例子中的100字节chunk），并不会释放给内核，而是进入到glibc的bin中，也叫做`freelist`。  
当下次申请内存时，会从bin中查找合适的chunk，而只有当没有合适的大小时，才向内核申请。

所以bin用来保存free过的chunk, 基于chunk的大小分为
- Fast bin
- Unsorted bin
- Small bin
- Large bin

但不同的bin都使用链表来管理这些chunk

## 测试方法
获取程序运行时的pid, 假设为949293, 那么通过以下命令可以获取程序的内存结构
```
cat /proc/949632/maps
# address           	  perms offset  dev    inode   			 pathname
55f7b378a000-55f7b378b000 r--p 00000000 103:02 4727634                   /tmp/a.out (deleted)
55f7b378b000-55f7b378c000 r-xp 00001000 103:02 4727634                   /tmp/a.out (deleted)
55f7b378c000-55f7b378d000 r--p 00002000 103:02 4727634                   /tmp/a.out (deleted)
55f7b378d000-55f7b378e000 r--p 00002000 103:02 4727634                   /tmp/a.out (deleted)
55f7b378e000-55f7b378f000 rw-p 00003000 103:02 4727634                   /tmp/a.out (deleted)
55f7b402c000-55f7b404d000 rw-p 00000000 00:00 0                          [heap]
7f4808000000-7f4808021000 rw-p 00000000 00:00 0 
7f4808021000-7f480c000000 ---p 00000000 00:00 0 
7f480ea3d000-7f480ea3e000 ---p 00000000 00:00 0 
7f480ea3e000-7f480f241000 rw-p 00000000 00:00 0 
...
```
* address 是起始地址-结束地址，差值为空间大小
* perm 是指权限，分为rwx[p,s], [p,s]指最后一位可以是private或者shared, 这些能通过`mprotect`命令修改(c中的const实现)
* offset 仅对mmap的区域有效，指映射偏移量
* pathname 如果此区域是映射的文件，则显示文件名。如果映射的是匿名映射则显示空白, 匿名映射包括线程通过mmap申请的堆内存，而[heap]是main函数的堆 

所以通过计算address可以知道glibc申请的内存情况

## `malloc`的返回值有意义吗

在我本机（gcc (Debian 10.2.1-6) 10.2.1 20210110 12GB）上测试了一段程序
1. 如果一次性申请13GB，malloc返回失败
2. 如果每次只申请2GB但不释放，第10次申请（20GB）malloc返回成功
3. 如果每次只申请2GB但不释放且`访问`，第三次申请（6GB）malloc返回成功，但访问时OOM

所以说，malloc的返回值是没有太大意义的

```
void foo(size_t large) {
  char *buffer = (char *)malloc(large);
  if (buffer == NULL) {
    printf("error!\n");
    exit(EXIT_FAILURE);
  }
  printf("Memory allocated\n");
  //  for (size_t i = 0; i < large; i += 4096) {
  //    buffer[i] = 0;
  //  }
  printf("Travalse done\n");
}
int main() {
  // size_t large = 1099511627776;
  size_t large = 13958643712;
  for (int i = 0; i < 100; i++) {
    foo(2147483648);
  }

  return EXIT_SUCCESS;
}
```

## 其他

内存管理不仅存在应用程序（或者说glibc）中，Linux内核也有内存管理。也不只有glibc的内存malloc一种实现，还有

- dlmalloc – General purpose allocator
- ptmalloc2 – glibc
- jemalloc – FreeBSD 的libc实现, 相比其他第三方，这个比较流行
- tcmalloc – Google 为多线程场景开发
- libumem – Solaris

malloc是glibc对ptmalloc2的包装，而`ptmalloc2`是`dlmalloc`的重构，区别很大  

最后总结，glibc的malloc通过两个系统调用函数brk和mmap来向内核申请内存，但内存释放后不会归还内核，而是放到freelist等待再次被利用

## 引用

https://man7.org/linux/man-pages/man5/proc.5.html
https://sploitfun.wordpress.com/2015/02/10/understanding-glibc-malloc/comment-page-1/
https://lemire.me/blog/2021/10/27/in-c-how-do-you-know-if-the-dynamic-allocation-succeeded/
