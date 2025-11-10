---
draft: true
comments: true
date: 2021-10-15
tags: [rhce]
---

# rhce8 考试记录
-- 考试日期：7月19日

# 学习笔记

## 04/26 NetworkManager 配置
rhel8 安装后没有安装dhcpd，只有dhcpclient， 所以必须手动执行dhcpclient 才能获取ip。 如果要自动连接，需要配置NM。   
开始的时候以为直接设置NM接管网卡和设置自动连接即可，重启发现不行  
```
$ nmcli device set enp1s0 autoconnect yes managed yes

```
正确办法是新建一个连接，这样重启后才会自动获取ip
```
# 语法
nmcli connection add type ethernet con-name connection-name ifname interface-name

```
```
$ nmcli connection add type ethernet con-name con-hst ifname enp1s0
```

## 0506 rhce8学习笔记，修改root密码 
1. 在控制台启动，选择内核的时候，按e修改内核启动脚本
2. 在内核选项后面，即在quiet 后面追加 rd.break，按ctrl+x 继续启动
3. 进入系统后hostname为`switch_root`，使用`mount | grep sysroot ` 查看挂载选项，这时候是ro只读挂载
4. 执行`mount -o remount,rw /sysroot/` 重新挂载sysroot
5. 再次执行`mount | grep sysroot` 查看到挂载选项变为 rw 可读可写
6. 执行`chroot /sysroot` 切换，然后执行`passwd` 来修改root密码
7. 执行`touch /.autorelabel`，然后exit退出，然后logout 重启系统


## 0512 rhce8学习笔记，系统调优 tuned-adm 工具
tune 是redhat开发的调优工具，已经制定好了一些方案，通过tuned-adm list 可以查看各种方案
```
[root@servera tuned]# tuned-adm list
Available profiles:
- balanced                    - General non-specialized tuned profile
- desktop                     - Optimize for the desktop use-case
- hpc-compute                 - Optimize for HPC compute workloads
- latency-performance         - Optimize for deterministic performance at the cost of increased power consumption
- network-latency             - Optimize for deterministic performance at the cost of increased power consumption, focused on low latency network performance
- network-throughput          - Optimize for streaming network throughput, generally only necessary on older CPUs or 40G+ networks
- powersave                   - Optimize for low power consumption
- throughput-performance      - Broadly applicable tuning that provides excellent performance across a variety of common server workloads
- virtual-guest               - Optimize for running inside a virtual guest
- virtual-host                - Optimize for running KVM guests
Current active profile: virtual-guest

```
1. 可以看到有针对desktop powersave 等优化。当前处于virtual-guest 的优化方案。
2. tuned 是服务进程，用来监控和收集各个组件的数据，并依据这些数据来动态调整。
3. tuned-adm 是客户端，通过这个命令来配置

### 配置方法
1. 查看推荐, 当前是虚拟主机，所以推荐了virtual-guest
```
tuned-adm recommend
virtual-guest
```
2. 选择方案和确认
```
[root@servera tuned]# tuned-adm profile powersave
[root@servera tuned]# tuned-adm active
Current active profile: powersave

```
这样就修改到powersave 模式了

