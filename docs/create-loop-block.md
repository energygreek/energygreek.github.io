---
date: 2020-08-05
tags: [filesystem, linux]
---

# 在文件系统之上再创一个文件系统

例如在ext3的文件系统上创建一个xfs的文件系统，可以通过回环设备loop， 我们经常通过 `mount -o loop ` 来 mount一个iso文件
但mount 的选项总是ro的

```
mount: /mnt: WARNING: device write-protected, mounted read-only.
```


不仅如此， 先在当前文件系统dd出一个文件， 再绑定到loop设备上，然后mount 到某个目录后， 可以进行`读写`访问

```
[root@ha1 ~]# dd if=/dev/urandom of=file bs=1M count=2
2+0 records in
2+0 records out
2097152 bytes (2.1 MB, 2.0 MiB) copied, 0.0164121 s, 128 MB/s

[root@ha1 ~]# mkfs.ext3  file 
mke2fs 1.44.6 (5-Mar-2019)
Discarding device blocks: done                            
Creating filesystem with 2048 1k blocks and 256 inodes

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (1024 blocks): done
Writing superblocks and filesystem accounting information: done

[root@ha1 ~]# mount -o loop file  /mnt
[root@ha1 ~]# ls /mnt/
lost+found

[root@ha1 mnt]# echo abc > abc
[root@ha1 mnt]# ls
abc  lost+found

```
绑定loop设备和挂载是由mount 一个命令完成的。

## 手动绑定loop 

```
[root@ha1 ~]# losetup -f
/dev/loop1

[root@ha1 ~]# losetup -f file 
```
这样只是将文件绑定了loop设备，需要再挂载到文件目录`mount /dev/loop0 /mnt`

```
[root@ha1 ~]# ls /mnt/
abc  lost+found
[root@ha1 ~]# 

```

losetup -f 可以返回第一个未被使用的loop设备名

## 创建loop设备

有的系统默认创建了 loop0 .. loop7 的块设备，有的则是在需要的时候创建，比如mount iso的时候发现没有loop设备，则会创建

1. 手动创建loop设备通过 `mknode` 创建

```
mknode  /dev/loop0 b 7 0
mknode  /dev/loop1 b 7 0
...

mknode /dev/loop7 b 7 0
```

2. 如果8个loop0 .. loop8 设备都占用了， 可以再创建loop8

```
$sudo mknod /dev/loop8 b 7 8 

$ls -l /dev/loop8
brw-r--r-- 1 root root 7, 8 Jun 11 19:16 /dev/loop8

```
ps： /dev/loop* 是块设备

