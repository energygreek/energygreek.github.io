---
comments: true
date: 2025-04-24
tags: [android]
---

# adb remote forward

连接安卓设备后，通过adb连接实现远程桌面的方法

1. 首先跳板机连接安卓，这里是wifi方式，adb connect 192.168.0.32:5555
2. 启动隧道转发，ssh -vCN -L5038:localhost:5037 -L27183:localhost:27183 adb@SERVER -p PORT
3. 在另一台设备上使用scrcpy远程安卓 
```
user@client:~$ export ADB_SERVER_SOCKET=tcp:localhost:5038
user@client:~$ scrcpy --force-adb-forward -b2M -m800 --max-fps=15
```

## 对于不能有线连接设备的情况，通过下面命令开启无线adb

在adb shell中操作

```
# This line enables adbd to run as tcpip by default
root@device:~$ echo "service.adb.tcp.port=5555" >> /system/build.prop

# This line enables adbd to accept input from keyboard & mouse using scrcpy
root@device:~$ echo "persist.security.adbinput=1" >> /system/build.prop

# This line enables adbd
root@device:~$ echo "persist.service.adb.enable=1" >> /system/build.prop

# After that, just reboot the DEVICE
root@device:~$ reboot
```

## 参考
https://psabadac.medium.com/ssh-adb-9d92c676d8c0