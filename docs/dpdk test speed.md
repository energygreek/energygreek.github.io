---
draft: true
comments: true
date: 2025-05-22
tags: [dpdk]
---

# dpdk-testpmd to test NIC speed

启动，进入dpdk 交互模式

```sh
./dpdk-testpmd -l 0-3 -n 4 --vdev=net_tap0,iface=tap0 -a 0000:07:00.0 -- -i --port-topology=chained 

#确认
testmpd> show port summary all 

#设置转发
testpmd> set fwd io 

#启动转发
testpmd> start 
```

设置tap设备的ip
```sh
ip add add 10.9.108.211/24 dev tap0
ip route add 10.9.3.13 via 10.9.108.211 dev tap0
#启动
iperf -s
```
