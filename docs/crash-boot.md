---
title: crash boot
date: 2021-01-04
tags: [grub, linux]
---

# 前言

元旦过完回公司发现archlinux 系统无法启动了， 提示没有找到内核之类的错误。 虽然也没有找到根本原因，只知道是内核镜像丢失了， 这里记一下解决办法

# 现象

启动之后， 提示缺少内核， 需要先指定内核。 回车后进入grub 界面。 在grub 界面可以执行ls (hd*,gpt*)/ 来查看分区的文件  
hd0、 hd1 代表的硬盘编号， gpt1、 gpt2、 gpt3 代表分区， ls (hd0, gpt1)/ 表示查看第一个硬盘第一个分区的文件

```
ls (hd1, gpt2)
EFI  grub  initramfs-linux-fallback.img
```

我的boot分区是第二个硬盘的第二个分区， 可见确实没有 vmlinuz-linux 文件，也很奇怪

## 解决办法

没有 'vmlinuz-linux' 文件的话需要通过archlinux的U盘启动盘启动， 挂载分区后，arch-chroot 进入到坏系统  

执行重装linux 
```
pacman -S linux
```
执行grub 相关命令， 重建引导配置。 我的EFI单独分区了

```
grub-install --target=x86_64-efi --efi-directory=/boot/EFI --bootloader-id=Arch
grub-config -o /boot/grub/grub.cfg
```

`exit`退出坏系统，umount 再重启。 一定要umount 分区否则grub不会生效

## 没有umount 导致的grub不生效

因为没有umount，`exit`后直接重启， 发现依旧无法进入系统，还是进入了grub界面，好在linux内核镜像已经存在了，可以通过grub来配置启动  
所以这种情况也适用于系统ok，但引导损坏的情况。

### 解决办法

我的boot分区是(hd1,gpt2)

执行一下
```
set prefix=(hd1,gpt2)/grub/  # 指定实际的grub目录
set root=(hd1,gp2)
insmod normal
normal
```

此时grub会刷新， 继续执行
```
insmod linux
linux /vmlinuz-linux root=/dev/nvme0n1p2   # 设置内核
initrd /initd.img
boot
```

由于不知道nvme的命名方式，导致也挺麻烦， grub下可以看到uuid,但不能看到分区名称， 后来发现可以通过uuid方式指定root
```
insmod linux
linux /vmlinuz root=UUID=xxxxxxxxxxx
initrd /initd.img
boot
```

## 扩充

同样可以通过磁盘+分区的方式指定内核和initrd



