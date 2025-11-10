---
date: 2021-01-21
tags: [linux,rhce]
---

# rhel系列修改密码

考rhce8 栽在改密码了， 现在彻底弄明白

## 关于selinux

这是rhel和其他发行版的最大区别，也是我忽略的点。启用selinux 时，改密码后，额外要执行`touch /.autorelabel`， 新密码才能生效，而平时我使用centos一直是禁用selinux的。

## 启用selinux

selinux 默认状态是enforcing, 禁用时为disable, 通过`sestate` 查看状态。

### 编译'/etc/selinux/config'

修改selinux配置，从disable 到enforcing

### 执行两次下面动作

创建这个文件的意义是重新label selinux, 不仅是当修改selinux配置需要做，在重置root密码时也是需要
```
touch /.autorelabel
reboot
```

## 叙说改密

rhel5~8有两种方式重置密码，老版本为给linux启动参数加上'init=/bin/sh', 新版本为加`rd.break`  
老版本适用于centos8（已测）, 而新版本应该不支持rhel5，6（未测），下面是完整步骤

### 老版本

1. 进入grub界面，按e进入编辑启动参数，到linux 行，按ctrl+e或者end键到末尾，追加'init=/bin/sh'
2. 按ctrl+x继续，系统自动进入内存文件系统的根目录
3. 执行`/sbin/load_policy -i`来初始化selinux
4. 此时系统处于ro模式，执行`mount -oremount,rw /` 重新挂载根分区，使系统可写
5. `passwd` 设置root密码
6. 如果启用了selinux, 额外要执行`touch /.autorelabel`
7. 最后执行`exit`或者 `exec /sbin/reboot` 或者`exec /sbin/init`

### 新版本

1. 进入grub界面，按e进入编辑启动参数，到linux 行，按ctrl+e或者end键到末尾，追加'rd.break'
2. 按ctrl+x继续，系统自动进入root系统，此时真正的文件系统以ro挂载在/sysroot
3. 执行`mount -oremount,rw /sysrot` 重新挂载
4. 执行`chroot /sysroot` 进入系统
5. 执行`passwd`设置root密码
6. 如果启用了selinux, 需要额外创建文件`touch /.autorelabel`
7. 执行`exit` 退出，然后执行 `umount /sysroot` 来确保写入
8. reboot 来重新进入

## 通过rescue模式改密码

任何发行版都可以通过光盘引导来改密。如果熟悉archlinux安装的方式，就知道进入live os之后可以挂载原系统的磁盘  
然后chroot成为root用户，就能直接执行`passwd`来修改密码

## 通过修改镜像改密码

最暴力的方式就是用guestfish工具, 一条命令改密。当然得先获取镜像文件  
先切换到root用户，使用guestmount命令挂载分区，-i表示自动挂载
```
guestmount --add base.raw  -i /tmp/hm
```
然后chroot到挂载点
```
chroot /tmp/hm
```
然后改密
```
passwd
```

virt提供了简化版本，一条命令就能改密码
```
virt-customize -a centos8.qcow2 --root-password password:123456
```

可见镜像文件有多么不安全

## 最后记录一下vmware虚拟机镜像转换kvm镜像

由于收到的vmware镜像，又没有装vmware workstation，所以找到转换镜像的方法，也很方便
```
qemu-img convert  CentOS\ 5.vmdk  base-000001.raw       
```
但问题是不支持转换快照文件

## 引用
https://docs.openstack.org/image-guide/convert-images.html

