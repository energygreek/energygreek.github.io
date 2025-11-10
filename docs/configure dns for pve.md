---
draft: false
comments: true
date: 2025-08-28
tags: [pve,dns]
---

# 为pve配置dns

我用闲暇台式机安装了pve, 创了一些虚拟机和lxc容器来跑本地部署的应用，例如代码管理的gitea, web应用，nginx反向代理等。 反向代理容器要访问其他容器，也要从外部访问容器，例如我的笔记本。 正好最近工作遇到些dns的问题，于是想自己搭建一个dns服务器。用来实现从外面，从容器都能通过域名来访问。就如同docker compose 里能通过容器名称来访问其他容器一样。反代nginx可以用frp等技术实现公网访问。

## 实现效果
在笔记本上能访问 gitea.pve， 在nginx反代容器里也能访问 gitea.pve，但是前者得到10地址，后者得到192地址。

## 网络规划
pve在安装了自动创建了网桥，桥接物理网卡，网络10.8.10.0/24，外面可以直接访问。后我自己创建了虚拟网络 192.168.10.1/24。于是每个容器和虚拟机都有两个地址，我称之外部地址和内部地址。 pve 创建网络位置在 Datacenter-> pve -> system -> network

## 宿主机器上安装bind9和配置
在AI的帮助下，了解了bind9的view专门用来完成这样的事情。先修改文件 /etc/bind/named.conf.default-zones, 将原来的部分套上 'external view'，然后加上自定义根域名'pve'

```diff                                                     
+view "external" {   
// prime the server with knowledge of the root servers
zone "." {
        type hint;
        file "/usr/share/dns/root.hints";
};

...

+zone "pve" {
+        type master;
+        file "/etc/bind/db.pve.external";
+};

+};
```

修改文件 /etc/bind/named.conf.local，原本这文件是空的，现在添加 'internal view'

```diff
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

+acl "internal-nets" {
+    192.168.10.0/24;    // VM subnet
+    127.0.0.1;          // localhost
+};
+
+view "internal" {
+    match-clients { internal-nets; };
+
+    zone "pve" {
+        type master;
+        file "/etc/bind/db.pve.internal";
+    };
};

```

然后在/etc/bind/db.pve.external 中添加域名记录，ip是外部地址， 在/etc/bind/db.pve.internal添加域名记录，ip是内部地址, 内容如下
```
$TTL 86400
@   IN  SOA ns1.pve. admin.pve. (
        2025082701 3600 1800 1209600 86400 )
    IN  NS  ns1.pve.

ns1      IN A 192.168.10.1
pve      IN A 192.168.10.1
docker   IN A 192.168.10.101
frp      IN A 192.168.10.102
gitea    IN A 192.168.10.103
```

然后检查配置和更新配置
```sh
rndc reconfig pve
rndc reload
```

注意： AI说修改了域名记录，要增加SOA号，例如 2025082701 -> 2025082702

## dig 验证
* 在笔记本中dig @pve的ip 查询容器ip
* 在pve里dig @192.168.10.1 查询容器的ip

## 外部配合修改dns服务器
上面dig命令指定了dns服务器的ip, 为了让系统自动选择pve上的dns服务器来解析 *.pve 域名，要修改笔记本配置，使用的NetworkManager
```sh
nmcli connection modify WIFI_CONNECTION ipv4.dns-search "~pve" 
nmcli connection modify MI-MINI_5G_D300 ipv4.dns 10.8.10.77
nmcli connection modify MI-MINI_5G_D300 ipv4.dns-priority -1 
```

而我用了有以太网和wifi两个网络，我只修改了wlp5s0。通过前后 resolvectl status 对比，区别如下

```diff
Link 3 (wlp5s0)
-    Current Scopes: LLMNR/IPv4 LLMNR/IPv6 mDNS/IPv4 mDNS/IPv6
+    Current Scopes: DNS LLMNR/IPv4 LLMNR/IPv6 mDNS/IPv4 mDNS/IPv6
         Protocols: -DefaultRoute +LLMNR +mDNS -DNSOverTLS DNSSEC=no/unsupported
+       DNS Servers: 10.8.10.77
+        DNS Domain: ~pve
     Default Route: no
```

这样是做了dns转发，把*.pve的域名转发给10.8.10.77来处理。 效果如下
```
# ping gitea.pve                  
PING gitea.pve (10.8.10.130) 56(84) bytes of data.
64 bytes from gitea.local (10.8.10.130): icmp_seq=1 ttl=64 time=0.588 ms
64 bytes from gitea.local (10.8.10.130): icmp_seq=2 ttl=64 time=0.767 ms
64 bytes from gitea.local (10.8.10.130): icmp_seq=3 ttl=64 time=2.95 ms
```

但是我配置 systemd-resolved 的 search 功能不生效，只能全写域名， 不能像在pve容器里那样省略'.pve'。

## 内部配合修改dns服务器

pve创建容器时，默认拷贝宿主机的dns配置即/etc/resolve.conf，先将其修改成如下，search 的作用是简化域名 gitea.pve 成 gitea 
```
nameserver 192.168.10.1
nameserver 1.0.0.1
search pve local
``` 

这样新建的容器的dns自动用搭建好的。 在已创建的容器里找到dns选项，手动修改。效果
```
# ping gitea
PING gitea.pve (192.168.10.103) 56(84) bytes of data.
64 bytes from 192.168.10.103 (192.168.10.103): icmp_seq=1 ttl=64 time=0.022 ms
64 bytes from 192.168.10.103 (192.168.10.103): icmp_seq=2 ttl=64 time=0.046 ms
```
