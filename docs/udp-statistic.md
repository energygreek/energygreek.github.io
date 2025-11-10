---
date: 2023-11-03
tags: [debug, linux]
---

# udp 查看统计的方法

* /proc/net/snmp
* /proc/net/udp

除此之外，还有通常使用netstat/ss 来查看发送队列和接收队列的缓冲区占用。

### /proc/net/snmp

```sh
cat /proc/net/snmp | grep Udp\:
```
输出格式
Udp: InDatagrams NoPorts InErrors OutDatagrams RcvbufErrors SndbufErrors

* InDatagrams recvmsg调用时，值会增加。
* NoPorts 发送到没有监听的端口，值会增加
* InErrors 没有内存，或者checksum失败
* OutDatagrams 成功发送到IP层，注意这不代表包成功发送出去了，因为在IP层可能有错误，比如路由错误，会导致IP层的OutNoRoutes值增加

### /proc/net/udp

```
cat /proc/net/udp
```
输出格式
  sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode ref pointer drops

* sl hash slot for the socket
* local_address 16进制的ip.端口
* st socket state
* tx_queue 发送队列的内存大小
* rx_queue 接收队列的内存大小
* inode socket对应的inode，可以判断哪个进程打开了这个socket
* drops 只记录了接收方向被丢掉的包，不包括发送方向丢掉的包。


## 参考
https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-sending-data/#monitoring-udp-protocol-layer-statistics
