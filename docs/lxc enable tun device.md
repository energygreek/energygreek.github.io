---
comments: true
date: 2025-08-21
tags: [pve]
---

# lxc enable tun device
在lxc中运行tailscale时，默认使用tun模式，就是创建一个虚拟网卡，然后用路由规则将出去的流量都路由到tun的虚拟网卡中。但是lxc中默认没有'/dev/net/tun'，这时tailscaled 启动时会尝试执行'modprobe tun'，结果lxc的内核版本和宿主的不一致，所以导致下面的报错。解决办法有2个，一是使用tailscale的userspace networking（也就是sock5代理），二是将宿主机的tun设备绑定到lxc容器中。

```
Aug 21 15:20:22 debian-dockers tailscaled[22673]：Linux kernel version: 6.1.10-1-pve
Aug 21 15:20:22 debian-dockers tailscaled[22673]: is CONFIG_TUN enabled in your kernel? `modprobe tun` failed with: modprobe: FATAL: Module tun not found in directory /lib/modules/6.1.10-1-pve
Aug 21 15:20:22 debian-dockers tailscaled[22673]: kernel/drivers/net/tun.ko found on disk, but not for current kernel; are you in middle of a system update and haven't rebooted? found: linux-image-5.10.0-32-rt-am>
Aug 21 15:20:22 debian-dockers tailscaled[22673]: wgengine.NewUserspaceEngine(tun "tailscale0") error: tstun.New("tailscale0"): CreateTUN("tailscale0") failed; /dev/net/tun does not exist
```

## 使用userspace networking
修改/etc/default/tailscaled文件，修改FLAG为'FLAGS="--tun=userspace-networking"'，这样tailscaled就不会去创建tun设备，而只是暴露一个端口，供应用来使用。  

使用tailscale的方法
```
ALL_PROXY=socks5://localhost:1055/ HTTP_PROXY=http://localhost:1055/ http_proxy=http://localhost:1055/ ./my-app
```

另外也可以手动启动tailscaled
```
tailscaled --tun=userspace-networking --socks5-server=localhost:1055 --outbound-http-proxy-listen=localhost:1055 &
tailscale up --auth-key=<your-auth-key>
```

## 给lxc容器添加tun设备

将下面两行添加到容器配置文件，例如容器id为100, 则配置文件的路径是/etc/pve/nodes/lxc/100.conf
```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net dev/net none bind,create=dir
```

## lxc enable fuse
当lxc挂载webdav文件系统时，依赖fuse 驱动，可以在pve的lxc选项里添加feature。 [参考](https://forum.proxmox.com/threads/persistently-use-fuse-inside-of-lxc-container.95810/) 

## 参考

https://tailscale.com/kb/1112/userspace-networking
https://github.com/tailscale/tailscale/issues/6941