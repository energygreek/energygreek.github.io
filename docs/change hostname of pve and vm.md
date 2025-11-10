---
comments: true
date: 2025-08-28
tags: [pve]
---

# change hostname of pve and vm
以前总不明白，创建虚拟机时填写的域名 example.com 的作用，直到自己用pve创建了多个虚拟机，

## change domain of pve
一开始我直接在系统里用hostnamectl来修改，然后发现执行pct命令时，报错没有虚拟机的配置文件，原来pct用hostname来隔离配置
```
$ pct set 110 --hostname docker.pve                                                        
Configuration file 'nodes/NEW_HOSTNAME/lxc/110.conf' does not exist
```
看官方的[文档](https://pve.proxmox.com/wiki/Renaming_a_PVE_node), 正确的方法要改好几个文件，于是重新改回去了。


## change domain of lxc
这个开始在虚拟机里改，后来知道正确方法是用pct工具
```
pct set ID --hostname NEW_HOSTNAME
```

## 参考
https://virtualizeeverything.com/2021/11/12/how-to-change-the-hostname-of-a-lxc-using-command-line-proxmox-7/
