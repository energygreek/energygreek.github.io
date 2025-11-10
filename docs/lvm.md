---
date: 2020-10-22
tags: [linux,lvm]
---

# LVM 功能介绍

LVM 是在硬件和系统之间多加了一层，可以很方便地修改分区和扩容。 从效率上看肯定有所损失的，但是对于企业而言，灵活性也很重要  
LVM具有在线扩容和快照的重要功能

## 在线扩容

当磁盘容量不够或者需要替换磁盘时，就需要扩容, 如果是替换磁盘就再需要换出旧硬盘

### 扩容步骤

使用lvm中的pvmove 命令假如原来是使用pv 为/dev/sda1, 新的硬盘为/dev/sdb1  
1. 新建pv: # pvcreate /dev/sdb1 
2. 扩容vg：# vgextend vg1 /dev/sdb1 
3. 如果要调整分区大小，使用lvresize -r -S 新大小。 不带-r的话需要重新mount才能生效， -S 是表示最终大小，-s +/- 是调整大小

### 换出步骤

vgreduce 是将pv踢出vg, pvremove是删除pv标记

1. 把sda1 上数据迁移到sdb1上： # pvmove  /dev/sda1  /dev/sdb1 
2. vgreduce 从vg1踢出sda1盘 # vgreduce vg1 /dev/sda1  
3. pvremove 删除磁盘分区的pv 标记 # pvremove /dev/sda1 

### 磁盘换出是否需要停业务

LVM的在线扩容是不需要停止业务的，但是应该在业务闲的时候做。LVM在做pvmove的时候，会冻结旧分区，新的操作就需要做备份。 
pvmove有时候很慢，但是不能中断，否则出现很多遗留的分区。 

与pvmove类似的功能的还有磁盘镜像， 磁盘镜像的好处很明显，即使复制失败了，原数据依然存在。

但两个技术都要求能够“原子化”，所以不得不对业务有所影响，而且影响的是磁盘IO， 无法通过降低业务的NICE值来减弱这种影响


## LVM 快照功能

当系统是安装在LVM文件系统之上，那么当需要进行高危操作的时候，可以先对分区进行快照备份，避免操作失败又无法恢复。
公司有一台centos7的自用系统， 需要升级到Centos8 以支持 Docker buildx。
虽然因操作到无法运行yum,dnf 时宣布失败，但好在有snapshot备份， 恢复后完好如初，太好用。

### 步骤 

1.  挂载如下

```bash
$ lsblk 
NAME                  MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda                     8:0    0   477G  0 disk  
├─sda1                  8:1    0   200M  0 part  /boot/efi
├─sda2                  8:2    0     1G  0 part  /boot
└─sda3                  8:3    0 475.8G  0 part  
  ├─centos-root       253:2    0    50G  0 lvm   /
  ├─centos-swap       253:4    0  15.7G  0 lvm   [SWAP]
  ├─centos-docker     253:18   0   150G  0 lvm   /var/lib/docker

```

根分区为50G, 而卷组还剩余160G,完全足够生成一个镜像，50G并未完全使用。
```
$ sudo vgs
  VG     #PV #LV #SN Attr   VSize   VFree   
  centos   1   4   0 wz--n- 475.74g  160.05g
  data     1  13   0 wz--n-  <7.28t <721.91g

```

2.  先对根分区进行快照，注意是`根分区`
```
# lvcreate -s -n <snapshot_name> -L <size> <logical_volume>

$ lvcreate -s -n backup -L 50G  /dev/centos/root

```
这里指定了快照的大小为50G, 理论上是要比源分区大， 这里也可以直接设置100G, 系统会自动调整大小到合适。

3. 进行一顿操作后， 进行恢复, 提示根分区正在使用， 重启后生效。

```
$ lvconvert --mergesnapshot /dev/centos/backup 
  Delaying merge since origin is open.
  Merging of snapshot centos/backup will occur on next activation of centos/root.

```

### 保存 

如果需要对快照进行备份到其他位置， 可以直接将快照分区挂载，然后通过tarball 来压缩。

```
tar -cvzf backup.tar.gz /mnt/lv_snapshot
```

还可以直接通过rsync 将挂载的snapshort 传输到其他服务器上

```
rsync -aPh /mnt/lv_snapshot  <remote_user>@<destination_server>:<remote_destination>
```

### 快照回滚

lvm的snapshort 也是写时复制的（copy on write），所以创建快照操作非常快， 
当分区进行删除和修改操作的时候， lvm文件系统会拷贝文件到其他地方保存。

### 问题

lvm快照不等于备份， 仅有快照是不能恢复的
