---
date: 2022-05-23
tags: [mdns,network]
---

# 使用systemd-resolved的mdns

之前的博客中使用了avahi来广播本机的hostname和服务，这对于打印机和其他无屏设备非常重要。而较新版本Linux发行版中systemd-networkd+systemd-resolved同样能实现此功能

## Network Manager

目前不少发行版默认的网络管理工具是NetworkManager, 需要先将其关闭和卸载，安装systemd-networkd包

## 配置systemd-networkd

networkd 中可以配置网桥，tun等设备，也可以直接在其中配置dns和dhcp等，在其中加入MulticastDNS=yes和Multicast=yes来启用mDns
```
[Match]
Name=eth0

[Network]
Address=192.168.4.42/24
DNS=114.114.114.114
Gataway=192.168.4.1
MulticastDNS=yes

[Link]
Multicast=yes
```

## 配置systemd-resolved

systemd-resolved功能类似与dnsmasq, 他会继承systemd-networkd中的dns信息，也会读取hostname用来广播，一般不用修改任何配置，之需要将其stub文件链接到/etc/resov.conf, 这样让系统的dns请求发到systemd-resolved

```
# ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
# ls -l /etc/resolv.conf
lrwxrwxrwx. 1 root root 37 May 22 22:03 /etc/resolv.conf -> /run/systemd/resolve/stub-resolv.conf
```

## 确认效果

通过`resolvectl status`命令可以查看resolve配置，如下全局的MulticastDNS是启用的，而networkd中的配置是per link的，也处于启用状态，Scopes里显示了mDNS,说明其启用了  

```
      Current Scopes: DNS LLMNR/IPv4 LLMNR/IPv6 mDNS/IPv4 mDNS/IPv6
```

```
[root@netpanc ~]# resolvectl status
Global
       LLMNR setting: yes
MulticastDNS setting: yes
  DNSOverTLS setting: no
      DNSSEC setting: allow-downgrade
    DNSSEC supported: yes
          DNSSEC NTA: 10.in-addr.arpa
                      16.172.in-addr.arpa
                      168.192.in-addr.arpa
                      17.172.in-addr.arpa
                      18.172.in-addr.arpa
                      19.172.in-addr.arpa
                      20.172.in-addr.arpa
                      21.172.in-addr.arpa
                      22.172.in-addr.arpa
                      23.172.in-addr.arpa
                      24.172.in-addr.arpa
                      25.172.in-addr.arpa
                      26.172.in-addr.arpa
                      27.172.in-addr.arpa
                      28.172.in-addr.arpa
                      29.172.in-addr.arpa
                      30.172.in-addr.arpa
                      31.172.in-addr.arpa
                      corp
                      d.f.ip6.arpa
                      home
                      internal
                      intranet
                      lan
                      local
                      private
                      test

Link 2 (eth0)
      Current Scopes: DNS LLMNR/IPv4 LLMNR/IPv6 mDNS/IPv4 mDNS/IPv6
       LLMNR setting: yes
MulticastDNS setting: yes
  DNSOverTLS setting: no
      DNSSEC setting: allow-downgrade
    DNSSEC supported: yes
  Current DNS Server: 114.114.114.114
         DNS Servers: 114.114.114.114
```

在其他设备上ping可以成功

```
PING netpanc.local (192.168.4.43) 56(84) bytes of data.
64 bytes from netpanc.local (192.168.4.43): icmp_seq=1 ttl=64 time=0.350 ms
64 bytes from netpanc.local (192.168.4.43): icmp_seq=2 ttl=64 time=0.799 ms
64 bytes from netpanc.local (192.168.4.43): icmp_seq=3 ttl=64 time=0.733 ms
```

## 参考

https://wiki.archlinux.org/title/Systemd-resolved#mDNS
