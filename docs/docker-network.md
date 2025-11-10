---
date: 2023-07-21
tags: 
  - docker
  - network
---

# docker network

记录一次docker的网络问题，以及学到的知识。

## 背景

公司用docker容器替代虚拟机来隔离，用来ssh登录以及运行不同的服务。容器连接了双网络，一个用来访问同宿主机上的容器，另一个用来从其他主机访问。

## 问题

宿主机重启之后，容器无法访问。需要手动改路由。

## 解决办法

每次修改路由麻烦，后面发现两个网络都是macvlan时，存在无法没有网络。


## 创建桥接物理网卡
这是需要指定parent为物理网卡，既然是桥接物理网卡，就能从其他宿主机访问了。
```
docker network create -d bridge   --subnet=10.9.3.0/24   --gateway=10.9.3.1   -o parent=ens39f0 public_net_bridge
```

```
docker network create -d macvlan  --subnet=10.9.3.0/24   --gateway=10.9.3.1   -o parent=ens39f0 public_net_macvlan
```
这两个由于使用的同一个网段/网卡，所以不可能同时存在。也许可以通过指定不同的ip-range来实现吗？


## 创建桥接虚拟网卡
这种网络就不能从非宿主机的其他主机访问，只能在同宿机下的容器之间访问。我称之为私有网络 
```
docker network create -d bridge   private_net_bridge
```

```
docker network create -d macvlan  private_net_macvlan
```
这里没有指定网络地址

## iperf测速
发现无论桥接物理网卡还是虚拟网卡，macvlan都要比bridge快

- 私有网络
bridge 模式分别测得: 20.4 Gbits/sec 22.6 Gbits/sec 23.0 Gbits/sec
macvlan 模式分别测得: 26.1 Gbits/sec 24.4 Gbits/sec 26.7 Gbits/sec

- 桥接物理网卡
macvlan: 25.7 Gbits/sec 25.5 Gbits/sec 27.3 Gbits/sec
bridge: 20.2 Gbits/sec 20.4 Gbits/sec 23.2 Gbits/sec


## 顺便说下容器连接多网络的方法
容器启动的时候，只能指定一个网络, 然后启动之后，可以通过命令来连接多网络，可以在启动时设置--restart=alway,这样重启主机后自动连接双网络
```
docker network connect private_net_bridge test_container
```

## 杂项
网络配置挺复杂，macvlan 下面还分
* Bridge mode, 就是常见的模式，走宿主机的网卡
* 802.1q trunk bridge mode, 先在物理网卡上创建子网卡

### IPvlan

除了bridge macvlan之外，还有IPvlan, 这种模式下，所有的容器都使用相同的mac地址，所以如果容器使用dhcp来获取ip地址，则可能有问题。 访问其他容器时，可以配置来使用macvlan模式。

### overlay

docker network driver 除了上面的三种模式之外，还有overlay, 用于连接不同主机的网络，可以跨宿主机访问容器。

## 宿主机网络类型
我通常会创建bridge并将物理网卡设为从卡，然后创建虚拟网桥，用于访问虚拟机。而除了bridge, 还有bond,另外还有team。总而言之，bridge可以使不同的虚拟机（client）使用网卡的虚拟化功能，而bond可以将不同的网卡绑定，实现负载均衡。如果两个网卡分别有1M带宽，那么bond后的网卡则有2M带宽。下面的链接先bonding,然后在bonding网卡上创建bridge网卡。[参考](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/networking_guide/sec-vlan_on_bond_and_bridge_using_the_networkmanager_command_line_tool_nmcli)

