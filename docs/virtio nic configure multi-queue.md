---
draft: false
comments: true
date: 2025-08-26
tags: [virtio]
---

# virtio nic configure multi-queue
virtio虚拟出来的网卡默认是单队列的，ip link 显示如下。 多核系统下，单队列要要阻塞收发， 影响性能， 多队列可以并发，内核也支持（TUN/TAP）。这里记录开启多队列的方式。 因为virtio的限制，队列最大256个。
```
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:16:b5:50 brd ff:ff:ff:ff:ff:ff
```

## libvirt 或者 virsh 修改网卡xml文件

```diff
<interface type='network'> 
      <source network='default'/> 
      <model type='virtio'/> 
+     <driver name='vhost' queues='N'/> 
</interface> 
```

## qemu 

```
qemu-kvm -netdev tap,id=hn0,queues=N -device virtio-net-pci,netdev=hn0,vectors=2M+2 ...
```

## openstack

```
openstack image set <IMG_ID> --property hw_vif_multiqueue_enabled=true
```

## 验证

ip link 的输出多了**mq**
```sh
$ ip link
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000    

$ ethtool -l enp1s0
Channel parameters for enp1s0:
Pre-set maximums:
RX:             0
TX:             0
Other:          0
Combined:       256
Current hardware settings:
RX:             0
TX:             0
Other:          0
Combined:       40
```

## 参考
https://docs.paloaltonetworks.com/vm-series/10-1/vm-series-deployment/set-up-the-vm-series-firewall-on-kvm/performance-tuning-of-the-vm-series-for-kvm/enable-multi-queue-support-for-nics-on-kvm
https://cloud.vk.com/docs/en/computing/iaas/how-to-guides/vm-multiqueue
https://www.linux-kvm.org/page/Multiqueue