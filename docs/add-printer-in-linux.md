---
title: add printer in linux
date: 2020-07-24
tags: [tool]
---
# linux 添加打印机

记一次arch 上添加打印机的过程， 公司有一个Ricoh c5503 的打印机，ip `192.168.4.99`
通过nmap这个地址可以发现开了不少端口

```bash
PORT     STATE SERVICE
21/tcp   open  ftp
23/tcp   open  telnet
80/tcp   open  http
139/tcp  open  netbios-ssn
514/tcp  open  shell
515/tcp  open  printer
631/tcp  open  ipp
7443/tcp open  oracleas-https
8080/tcp open  http-proxy
9100/tcp open  jetdirect
```

## 启动avahi-daemon 服务

这个服务提供mDNS功能， 可以发现局域网内的主机提供的服务，例如打印服务，ssh。 各个主机通过`224.0.0.251` 地址多播自己提供的服务  
主机也可以配置自己想广告的服务，这是用户可选的。 具体看wiki 

通过执行一下命令可以看到打印服务
```
 avahi-browse --all --ignore-local --resolve --terminate 
```

找到有用的信息如下:
```
+    br0 IPv4 RICOH MP C4503 [002673734B1D]                 Web Site             local
=    br0 IPv4 RICOH MP C4503 [002673734B1D]                 Web Site             local
   hostname = [RNP002673734B1D.local]
   address = [192.168.4.99]
   port = [80]
   txt = ["path=/"]

```

## 安装cupsd

启动`cups-browsed.service`服务后， 可以通过网页也配置打印机， 包括分享自己的打印机和添加别人的打印机。 
我这里需要添加打印机服务

1. 浏览器输入`http://localhost:631/` 可以打开网页  
2. administrator -> add printer 输入root和密码  
3. 有以下选项
```
Backend Error Handler
Internet Printing Protocol (ipp)
LPD/LPR Host or Printer
Internet Printing Protocol (ipps)
AppSocket/HP JetDirect
Internet Printing Protocol (http)
Internet Printing Protocol (https)
Windows Printer via SAMBA 
```
4. 我这里选择的`AppSocket/HP JetDirect`， 因为nmap显示是支持`jetdirect`的。 
5. 输入地址`socket://192.168.4.99:9100`
6. 输入自定义的打印机名称和路径`/`

至此，浏览器打印页面就可以看到添加的打印机了

## 分享打印机
没有分享的时候，只有本机可以使用，当想要给手机使用时，可以勾选分享。此时CUPS会使用mdns广播打印机服务到主机所处的网段，所以手机也必须在这个网段里。
![[cups_share.png]]

## 更新

开始时我走的ipp协议，打印机地址为'ipp://192.168.4.99:631/ipp', 后来连不上打印机  

通过查端口发现有维护人员修改了打印机端口信息, 631和80端口都被封了，可能是出于安全考虑  
```
PORT      STATE    SERVICE
21/tcp    open     ftp
23/tcp    open     telnet
80/tcp    filtered http
139/tcp   open     netbios-ssn
514/tcp   open     shell
515/tcp   open     printer
631/tcp   filtered ipp
7443/tcp  open     oracleas-https
8080/tcp  open     http-proxy
9100/tcp  open     jetdirect
65000/tcp open     unknown
```
631没有开放，参考windows的同事配置，发现他们用的9100端口(9100是走socket协议)，所以需要重新添加打印机  
新的网络地址即为'socket://127.0.0.1:9100'


## 更新2
之前是自己搓的桌面环境，需要手动在CUPS的web界面添加打印机。而自从使用了完整桌面GNOME发现非常简单。GNOME自带有打印机配置工具，只需要填入打印机的IP（也许由于不在一个网段），就会自动检测支持的协议并配置好。

![[Pasted image 20240410175417.png]]