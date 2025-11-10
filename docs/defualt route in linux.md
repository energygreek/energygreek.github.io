---
draft: false
comments: true
date: 2025-07-10
tags: [network,route]
---

# defualt route in linux
默认路由非常重要，当对端地址在路由表中查不到从哪个网口出去时，就走默认路由对应的网卡。

## 即使是只有一张网卡，也应该配默认路由
最近遇到问题，使用命令`ip addr add 10.9.3.211/24 dev enp1s0`给虚拟机加ip后，路由表如下:
```sh
#ip route
10.9.3.0/24 dev enp1s0 proto kernel scope link src 10.9.3.211 
```
能ping网关和其他同网段主机，但ping不同其他网段的主机，显示'network unreachable'， 反向也不行。使用命令`ip route add default via 10.9.3.1 dev enp1s0`配置默认路由，问题就解决了。而通过网卡配置和网卡启动脚本例如`ifup`，或者网络管理工具NetworkManager，是会自动添加默认路由的，所以不存在这个问题。

## 拓展1，路由Metric
所有匹配不到的包就走默认路由指定的网关，如果存在多个网卡，可以有多个默认路由如下，这时候靠metric来指定优先级，metric在配置里指定

```
default via 10.14.160.1 dev enp1s0 proto static metric 90 
default via 10.14.162.1 dev enp10s0 proto static metric 100 
default via 10.14.161.1 dev enp9s0 proto static metric 100 
```
这时10.14.160.0/24网段的包走enp1s0，10.14.162.0/24走enp10s0， 10.14.161.0/24走enp9s0，因为enp1s0的metric最小，优先级最高，__其他网段的包走enp1s0__。  

### 手动再添加路由，优先级比默认路由高
例如指定1.1.1.1走非默认路由，
```
ip route add 1.1.1.1 via 10.14.161.1 dev enp9s0 
```
这时ping 1.1.1.1就不走默认路由了。

### lookup 设置表的优先级
上面的手动添加路由可以指定非3个网段的包走非默认路由，但是无法超越同网段的优先级，意思就是无法指定10.14.161.0/24的包走10.14.160.1这个网关。这时就需要创建新的路由表，通过表的优先级来实现

```
echo "200 custom1" | sudo tee -a /etc/iproute2/rt_tables
sudo ip route add 10.14.162.100 via 10.14.160.1 dev enp1s0 table custom1
sudo ip rule add to 10.14.162.100 lookup custom1
```

不过似乎大部分情况没意义，我遇到有意义的情况是使用了tailscale exit node时，所有包都走了隧道。如果希望本地某个主机不走隧道，就需要lookup 自定义路由表。