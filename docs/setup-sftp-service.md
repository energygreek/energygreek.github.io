---
date: 2021-06-25
tags: [linux,filesystem]
---

# setup sftp service

## curlfs 和 sshfs 客户端
在debian上，之前一直用curlfs工具，将远程目录mount到本地目录。系统升级之后发现这个包没有了， 原来因为ftp不安全，所以现在推荐sshfs。

##  reference

https://www.linuxtechi.com/configure-sftp-chroot-debian10/
