---
comments: true
draft: true
date: 2024-12-26
tags: [systemd, unix-domain]
---

# systemd socket 服务
一直非常喜欢用unix domain作为nginx和反代服务之间通讯方式， 将能支持的服务软件都尽量走unix domain。因为众所周知比127.0.0.1快，之前也测试过的确如此。


## unix domian socket in systemd

### 设置权限
SocketMode=0666
SocketUser=root
SocketGroup=root



## 性能测试



### c

```
recieve 440014 message
```