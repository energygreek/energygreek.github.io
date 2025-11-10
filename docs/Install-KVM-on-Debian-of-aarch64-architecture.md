---
date: 2020-10-30
tags: [aarch64]
---

# Install KVM on Debian of aarch64 architecture

公司有台arm64位的设备，安装的银河麒麟系统，经验证就是debian 9。  
现在需要运行更多arm64虚拟机。

# 安装步骤

## 配置免密sudoer 

这里记一个坑， 第一次配置时拼写错误，导致sudo 命令执行失败。需要重置密码，所以很麻烦。   
然后推荐使用 `sudo visudo` 命令来修改sudoer配置文件，这个命令在编辑结束后校验配置文件，避免出现编辑错误导致无法使用sudo的情况。  

```
kylin@Kylin:~$ cat /etc/sudoers.d/kylin 
kylin	ALL=(ALL:ALL) NOPASSWD:ALL
```

## 配置软件源 

第一个是本地iso, 下面的是中科大的镜像源。 每个都设置了trust免校验gpg

```
deb [trusted=yes] file:///mnt/iso/ juniper main multiverse restricted universe
deb [trusted=yes] http://ftp.cn.debian.org/debian/ stretch main contrib non-free
deb [trusted=yes] http://ftp.cn.debian.org/debian/ stretch-updates main contrib non-free
deb [trusted=yes] http://mirrors.ustc.edu.cn/debian-security/ stretch/updates main contrib non-free
deb [trusted=yes] http://ftp.cn.debian.org/debian/ stretch-backports main contrib non-free 
```

##  安装虚拟工具

安装 qemu, libvirt, virt-manager

```
sudo apt install  qemu  libvirt virt-manager
```

## libvirt 没有网络

执行`virsh net-list -all` 发现default 网络处于未激活状态， 于是执行 `systemctl status libvirtd` 发现

```
10月 30 15:46:27 Kylin libvirtd[16741]: 2020-10-30 07:46:27.262+0000: 16764: error : virFileReadAll:1420 : 打开文件 '/sys/class/net/virbr0-nic/operstate' 失败: 没有那个文件或目录
10月 30 15:46:27 Kylin libvirtd[16741]: 2020-10-30 07:46:27.262+0000: 16764: error : virNetDevGetLinkInfo:2530 : unable to read: /sys/class/net/virbr0-nic/operstate: 没有那个文件或目录
10月 30 15:46:54 Kylin libvirtd[16741]: 2020-10-30 07:46:54.710+0000: 16745: error : virFirewallApply:916 : 内部错误：Failed to initialize a valid firewall backend
```
需要安装以下组件，然后重启libvirtd，这是因为libvirt的网络依赖这几个组件来创建nat 网络

```
sudo apt install ebtables iptables dnsmasq
systemctl restart libvirtd
```

## virt-manager 提示aarch64 安装 uefi 错误

直接安装 qemu-efi
```
apt install qemu-efi
```

## virt-manager 允许非root用户访问

因为当以普通用户运行 virt-manager 时，qemu的连接指向非qemu:///system， 这时看到的虚拟机和root看到的不一样，所以必须使用sudo运行。
通过修改配置来启用普通用户也访问`qemu:///system`， 将uri_defualt 的注释去掉。

```
vim /etc/libvirt/libvirt.conf 

#
# These can be used in cases when no URI is supplied by the application
# (@uri_default also prevents probing of the hypervisor driver).
#
uri_default = "qemu:///system"
```


# 这样就可以使用virt-manager来创建虚拟机了，跟x86的使用一样。

顺便记一下vnc的配置， 因为机器在机房里， 通过tigervnc来使用桌面。  
另外提示一下，virt-manager 配置好后，可以直接连接远程的virt-manager,非常方便。

使用tigervnc 连接需要在远程主机安装 tigervnc-server 和 tigervnc-password。
使用普通用户执行 `tigervnc-password ` 设置访问密码，然后直接执行`tigervnc-server`。

这时候vnc只接受本地连接， 如果远程访问，要使用ssh 本地转发来实现。 这也是vnc推荐的方式。

现在本地允许本地转发命令， 监听本地端口 9302, 转发到远程的 9300
```
ssh -L 9302:localhost:9300   -N -f  ssh-kylin
```

然后执行 `vncviewer  127.0.0.1:9302` ，输入密码就能打开远程桌面了。


 
