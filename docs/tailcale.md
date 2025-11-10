---
draft: false
comments: true
date: 2025-04-30
tags: [tailcale,pve]
---

# tailscale 使用记录
tailscale 免费最大提供10个节点，足够个人使用。在如今大内网的时代，非常方便组建子网，实现家中和公司互联互通。

## 添加DERP节点
如果两个节点有一个有公网IP, 或者能打洞（NAT1. NAT2），那么两个节点可以直连，速度很快。否则要通过DERP节点转发，但是tailscale在大陆没有DERP节点，HK的DERP节点都有300+的延迟，完全没法用。 所以只能自己搭建或租赁一个在大陆的私人搭建的节点。

## 出口节点
出口节点A是流量可以从这个节点出去，比如节点B可以从A查询IP地址时，显示的ip是节点A的公网ip 而不是B的。
### 转发到出口节点
在B节点上设置流量都从节点A出去
tailscale up --exit-node=<A-Exit-Node-IP>

### 添加转发例外
启动转发后，所有流量都会从出口节点走。如果单机就没事，但是局域网多台电脑需要交互，例如从局域网主机C访问主机B, 由于主机B的流量都从节点A出去了，会导致主机C收不到请求答复。结果是C 无法ping B, C也不能ssh登录B。 所以要添加例外

将来自本地局域网主机C的流量不被转发到出口节点。例如路由器的子网是192.168.10.0/24，主机C是192.168.10.111, 本机ip是192.168.10.100
```sh
# ip rule add to <your-local-network> lookup main
ip rule add to 192.168.10.111/32 lookup main
```

