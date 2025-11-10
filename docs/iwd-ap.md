---
date: 2023-11-08
tags: [linux, iptables]
---

# 使用iwd创建ap热点
windows中有很多软件能创建热点，现在Linux也可以创建热点了，下面记录方法

## 确认网卡支持AP
使用`iw list`查看wifi网卡信息，如果在Supported interface modes能找到AP，说明网卡支持创建热点。

## 使用iwd
之前研究的wifi棒子，都是用wpa_supplicant来创建热点。而在我自己的电脑里一直使用的intel的iwd。
1. 先修改iwd的配置文件/etc/iwd/main.conf
```
[General]
EnableNetworkConfiguration=true
```
2. 在iwd保存密码的目录创建ap子目录
```
mkdir -p /var/lib/iwd/ap/
```
3. 在ap目录创建配置文件名字如main.ap，内容如下
```
[Security]
Passphrase=password123

[IPv4]
Address=192.168.250.1
Gateway=192.168.250.1
Netmask=255.255.255.0
DNSList=8.8.8.8
```
4. 执行iwctl命令进入到iwd交互模式
```
[iwd]# device wlan0 set-property Mode ap
[iwd]# ap wlan0 start-profile main
[iwd]# 
```
5. 启动内核转发
```
# sudo sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1
# sudo iptables -t nat -A POSTROUTING -s 192.168.250.0/24 -j MASQUERADE
```

我做完上面的步骤后，发现wifi连接不上，也有报错。然后重启了iwd服务，发现没有了main这个热点，所以这个热点不会自动启动。在执行4步骤后，就能正常连接wifi了。

## ad-hoc 和 ap 的区别
ad-hoc是去中心化的组网方式，各连接节点可以直接通讯，而有些终端可能不支持。AP模式又叫做infrustructure Mode，就是类似传统的路由器模式，所以兼容性更好。

## 使用总结
这个ap不稳定，长时间使用后没有网络， 需要重建ap， 不推荐。最终还是装了个便宜路由器MI-mini。老旧路由器的性能差，当路由（NAT）的丢包率太高， 但当桥接入口（交换机）正常。

## 参考
https://iwd.wiki.kernel.org/ap_mode
