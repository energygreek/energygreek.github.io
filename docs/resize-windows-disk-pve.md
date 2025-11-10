---
date: 2024-04-25
tags:
  - pve
  - vnc
  - gparted
---
# pve中给windows扩容
当初创建windows时给的40G硬盘，就一个分区，后来装了一些软件，发现不够用了，就第一次扩容，然后加了另一个分区。后来有些东西必须放在C盘里，这次扩容就比较复杂些。

## 第一次扩容
pve的web界面点击硬盘的resize可以修改硬盘大小，这个硬盘实际上是一个lvm块设备，而对于虚拟机Windows而言就是硬盘变大了。

硬盘加大后，进入windows磁盘管理，发现Windows已经有3个区：efi，C盘，恢复分区。为了方便直接加了一个D盘。

备注：pve的虚拟机安装在块设备上，而qemu是用文件（raw or qcow2），所以qemu的虚拟机可以复制移动，不过pve支持异地部署。

## 第二次扩容
这次扩容是为了增加C盘大小，而C盘后面有恢复分区和D分区。不得不在gparted 中执行:
1. 删除D分区
2. 移动恢复分区到磁盘末尾
3. 拓展C盘大小。
gparted操作很方便，不过windows启动不成功，通过windows安装iso修复之后正常了，恢复分区也还在。
## virtual vnc 可视化打开gparted
因为pve没有连接显示器，只能用虚拟vnc，远程连接到pve桌面。

## 安装tigervnc服务端
pve只需要安装服务端，tigervnc-standalone-server。本地电脑安装客户端，选择很多例如 remmina。
## 启动
创建密码后启动vnc 服务端失败，提示没有桌面。原因是没有安装桌面环境更没有启动桌面，好在tigervnc 支持指定可视化进程（xVNC就不支持这种方式）。
```
tigervncserver -xstartup /usr/sbin/gparted  --  /dev/dm-15
```

这里直接给的gparted可执行文件路径和参数，`--`表示后面的参数是给gparted的，不是给tigervncserver的，/dev/dm-15是要操作的windows磁盘，也能在gparted界面里面再选。

## remmina设置窗口大小
tigervncserver默认启动1024x768的窗口，当内容多时不方便。重新启动
```
tigervncserver -geometry 1920x1080 -xstartup /usr/sbin/gparted  --  /dev/dm-15
```

但是重新连接发现窗口还是很小，以为参数不生效。后来发现这时候窗口右下角可以拖动，手动拖成满屏

## gparted问题
启动gparted后出现没有光标，后面按了鼠标右键就恢复X光标了。arch wiki里面有解决[办法](https://wiki.archlinux.org/title/TigerVNC#:~:text=If%20no%20mouse%20cursor%20is%20visible) 