---
draft: true
date: 2021-05-13
tags: [linux,tmpfile]
---

# 使用systemd-tmpfiles来管理临时文件
vscode使用clangd作为c++的lsp非常好用，可以支持跳转、补全、clang-tidy、提示，但同时也对配置有要求，我的12Gi内存经常不够用而非常卡  
于是摸索了一下，发现有两个问题：
1. clangd会占用很多内存 
2. clangd在处理代码文件时会产生pch文件默认放到/tmp目录占用内存

这里会说怎么解决这2个问题，也顺便用到systemd-tmpfiles工具

## 问题

### 搜索过程

由于代码比较复杂，随便跳转几个文件就开始感到卡顿，一会就不能动了。切换X界面到终端, 执行`free`命令，可以看到shared和buff/cache两个部分占用很大内存
```
~ > free -h
               total        used        free      shared  buff/cache   available
Mem:            11Gi       7.1Gi       721Mi       1.0Gi       3.7Gi       3.1Gi
Swap:             0B          0B          0B
```

通过man可以知道这几项的意思, 可见share在我的系统指'/tmp', 因为tmpfs挂载在/tmp目录
```
shared 
Memory used (mostly) by tmpfs (Shmem in /proc/meminfo)

buffers
Memory used by kernel buffers (Buffers in /proc/meminfo)

cache  
Memory used by the page cache and slabs (Cached and SReclaimable in /proc/meminfo)

buff/cache
Sum of buffers and cache
```
而buff/cache指的缓存(cache)和内核(buff)占用, 通过`cat /proc/meminfo` 查看到buff占用不大，不是内核问题(因为我用的最新内核,所以抱有怀疑态度)

```
~ > cat /proc/meminfo  | grep "Buffers\|Cached"
Buffers:          308440 kB
Cached:          3297168 kB
```

所以主要是/tmp目录占用内存过大，以及clangd占用内存过大

### 解决办法

1. 对于clangd占用内存过大， 只有添加`-background-index=0`时，才能避免内存占用过大
2. clangd在分析代码文件的时候，产生大量pch文件, 这些都会占用内存

```
~ > ls /tmp/*.pch -lh
-rw-r--r-- 1  100M May 13 11:18 /tmp/preamble-133598.pch
-rw-r--r-- 1  104M May 13 11:17 /tmp/preamble-457bd6.pch
-rw-r--r-- 1  104M May 13 11:17 /tmp/preamble-465a46.pch
-rw-r--r-- 1  104M May 13 11:18 /tmp/preamble-99f5e5.pch
-rw-r--r-- 1  99M May 13 11:17 /tmp/preamble-b56fad.pch
```

## 使用systemd-tmpfiles-clean 来清除/tmp目录下的pch文件

### systemd-tmpfiles 介绍

systemd-tmpfiles 通过`systemd-tmpfiles-clean.timer`定时调用`systemd-tmpfiles-clean.service`, 来清理零时文件，且是默认启动的。  
虽然/tmp目录使用的内存挂载`tmpfs`（也有写发行版使用物理磁盘来挂载），在重启时会清空，但对于服务器这种常年不会重启的系统，就需要这样的机制清理内存/磁盘。  

timer默认的配置文件，`OnBootSec=15min`表示在系统启动15分钟后开始执行，而`OnUnitActiveSec=1d`表示在执行一次后，1天后再执行
```
~ > cat /usr/lib/systemd/system/systemd-tmpfiles-clean.timer
#  SPDX-License-Identifier: LGPL-2.1-or-later
#
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.

[Unit]
Description=Daily Cleanup of Temporary Directories
Documentation=man:tmpfiles.d(5) man:systemd-tmpfiles(8)

[Timer]
OnBootSec=15min
OnUnitActiveSec=1d
```


上面的timer会定时调用`systemd-tmpfiles-clean.service`这个服务

```
~ > sudo systemctl status systemd-tmpfiles-clean.service
○ systemd-tmpfiles-clean.service - Cleanup of Temporary Directories
     Loaded: loaded (/usr/lib/systemd/system/systemd-tmpfiles-clean.service; static)
     Active: inactive (dead) since Wed 2021-05-12 16:37:27 HKT; 22h ago
TriggeredBy: ● systemd-tmpfiles-clean.timer
       Docs: man:tmpfiles.d(5)
             man:systemd-tmpfiles(8)
    Process: 36637 ExecStart=systemd-tmpfiles --clean (code=exited, status=0/SUCCESS)
```

## systemd-tmpfiles 配置

上面是timer的设置，这个timer适合服务器，但并不建议修改它们。而是添加规则，再手动调用`systemd-tmpfiles --clean`
配置规则的方法可以查看`man tmpfiles.d 5`，但只建议在/etc 和 ～/ 目录添加规则, 前者对所有用户生效，后者只对自己生效

```
TMPFILES.D(5)                                                                           tmpfiles.d                                                                           TMPFILES.D(5)

NAME
       tmpfiles.d - Configuration for creation, deletion and cleaning of volatile and temporary files

SYNOPSIS
       /etc/tmpfiles.d/*.conf
       /run/tmpfiles.d/*.conf
       /usr/lib/tmpfiles.d/*.conf

       ~/.config/user-tmpfiles.d/*.conf
       $XDG_RUNTIME_DIR/user-tmpfiles.d/*.conf
       ~/.local/share/user-tmpfiles.d/*.conf
```

直接从/usr/lib/tmpfs.d/ 找合适的配置，拷贝到自己的目录再改

删除/tmp/*.pch文件的效果
```
~ > free
               total        used        free      shared  buff/cache   available
Mem:        12158548     7808240      854296     1010820     3496012     3019688
Swap:              0           0           0
~ > rm /tmp/*.pch
~ > free
               total        used        free      shared  buff/cache   available
Mem:        12158548     7804356     1169052      699936     3185140     3334460
Swap:              0           0           0
```

shared项少了许多, 再设置`background-index=0`后，内存占用大大减小

## 总结

使用systemd-tmpfs 来清除文件的方式不太适合这种场景, 但其本身是非常功能强大的管理工具
