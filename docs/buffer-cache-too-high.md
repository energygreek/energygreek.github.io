---
title: buffer-cache too high
date: 2021-04-15
tags: [memory,linux]
---

# Linux 调优

系统原厂商是不喜欢讨论系统调优的，一方面说起来没完没了，二来比较复杂，而且私以为调优即说明系统默认不够好？

而且SUSE的原厂[规定](https://www.suse.com/support/handbook/#donotsupport):

原理机制的介绍及系统调优并不在我们的技术支持范畴

这里是一点相关介绍

## buffer/cache 的作用和区别

buffer是用于存放将要输出到disk（块设备）的数据，而cache是存放从disk上读出的数据。二者都是为提高IO性能而设计的。                                                                                                                                                         
- buffer：缓冲将数据缓冲下来，解决速度慢和快的交接问题；速度快的需要通过缓冲区将数据一点一点传给速度慢的区域。                                                                                 
例如：从内存中将数据往硬盘中写入，并不是直接写入，而是缓冲到一定大小之后刷入硬盘中。                                                                                                           
A buffer is something that has yet to be "written" to disk.                                                                                                                                    
                                                                                                                                                                                               
- cache：缓存实现数据的重复使用，速度慢的设备需要通过缓存将经常要用到的数据缓存起来，缓存下来的数据可以提供高速的传输速度给速度快的设备。                                                      
例如：将硬盘中的数据读取出来放在内存的缓存区中，这样以后再次访问同一个资源，速度会快很多。                                                                                                     
A cache is something that has been "read" from the disk and stored for later use.                                                           

总之buff和cache都是内存和硬盘之间的过渡，前者是写入磁盘方向，而后者是写入内存方向

### 回收cache

```
drop_caches回收一下。
#sync;sync;sync
#echo 3 > /proc/sys/vm/drop_caches    
```
free增加300M

## swap 介绍

Swap意思是交换分区，是硬盘中的一个分区。内核将内存Page移出内存到swap分区(swap out)

swap通过 vm.swappiness 这个内核参数控制，默认值是60。`cat /proc/sys/vm/swappiness` 可以查看当前值  
这个参数控制内核使用swap的优先级。该参数从0到100。

设置该参数为0，表示只要有可能就尽力避免交换进程移出物理内存；                                                              
设置该参数为100，这告诉内核疯狂的将swapout物理内存移到swap分区。
注意：设置该参数为0，并不代表禁用swap分区，只是告诉内核，能少用到swap分区就尽量少用到，设置vm.swappiness=100的话，则表示尽量使用swap分区。                                                    
                                                                                                                                                                                               
这里面涉及到当然还涉swappiness及到复杂的算法。如果以为所有物理内在用完之后，再使用swap, 实事并不是这样。以前曾经遇到过，物理内存只剩下10M了，但是依然没有使用Swap交换空间，另外一台服务器，物理内存还剩下15G，居然用了一点点Swap交换空间。
其实少量使用Swap交换空间是不会影响性能，只有当内存资源出现瓶颈或者内存泄露，进程异常时导致频繁、大量使用交换分区才会导致严重性能问题。             

### 问题：何时使用swap

这个问题如上面说的，比较难说，理论上是当物理内存不够用的时候，又需要读入内存时，会将一些长时间不用的程序的内存Page 交换出去。                       
但是很多时候会发现，内核即使在内存充足的情况下也是使用到swap

### 问题: 那些东西被swap了？

可以看下面的测试

### 回收swap
 
swapoff 之后执行`sudo sysctl vm.swappiness=0` 临时让内核不用swapout

```
并把swap的数据加载内存，并重启swap 
#swapoff -a
#swapon -a
```
即把swap分区清空,  自测效果如下，内核版本`5.10.0-8-amd64`

```
               total        used        free      shared  buff/cache   available
Mem:        12162380     4911564     5605744      459364     1645072     6466572
Swap:        1000444      763040      237404
```

重启swap后
```
               total        used        free      shared  buff/cache   available
Mem:        12162380     5605800     4843176      524984     1713404     5707112
Swap:        1000444           0     1000444
```

可见，停用swap后，swap的used大部分到了mem的`used`，小部分到了Mem的shared

## 调优的一些有效工具

perf + flame火焰图: 查看运行耗时，可以查看函数调用耗时，如果是自己的程序，可以知道哪些函数需要优化
vmstat 查看磁盘io情况，使用`vmstat -t 3`命令，如果b状态的数字一直很大，那么说明磁盘阻塞严重，可能是磁盘坏了，可能是程序设计不合理

还有top，iperf等等
