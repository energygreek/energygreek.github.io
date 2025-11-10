---
date: 2020-11-05
tags: [android,lineage]
---

#  huawei unlock bootloader

解锁华为平板M3的BL锁，以及获取root权限

# 背景

我有一个华为平板M3，WIFI版, 型号BTV-W09，系统是emui5， 买来没什么用，最大的功能就是看视频。 偶尔发现一个app，LinuxDeploy, 可以在安卓上安装完整的Linux系统，而不是内置的阉割版， 前提是获得root权限。

# 步骤

全部步骤分4个：   
1. 解BL
2. 刷入recovery 也就是RTWP
3. 将root压缩包复制到平板，在RTWP下安装
4. 安装supersu 的apk
5. 可选刷入xposed框架，并安装xposed manager，步骤参考3,4

##  解锁

因为华为官方停止申请解锁BL的服务， 所以需要上淘宝找人帮你搞定。 解锁BL之后， 在关机状态按住电源和音量减，进入fastboot模式时，有红字提示unlocked


## 刷入rec

找到与设备型号对应的rec非常重要，因为型号不对会刷不进去，我尝试了很多个版本，最终在华为论坛找到了。 
有了rec后， 让手机处于fastboot状态， 连接手机到电脑，使用 `fastboot flash recovery rec` 来刷入 ，提书刷入成功之后，可能自动重启，如果没有重启，长按电源键强制关机。   
关机状态下， 按住电源键和音量+，进入recovery ， 能看到RTWP的界面说明输入成功，如果没有看到RTWP,而是进入华为官方的eRecovery 表示失败，又可能是被华为覆盖了。
一旦能进入RTWP, 那么RTWP会自动安装，以后就不用担心被覆盖的问题。

## 刷入root工具

将root.zip 拷贝到平板的储存卡目录，进入RTWP，点INSTALL,  然后选择root.zip 就会开始安装了， 安装后再安装supersu 的应用


这样就root成功了， 所以步骤很简单，找到对应机型的rec 很关键。 

## 刷入xposed 

xposed 很强大，但是xposed只是个框架，需要安装包来实现对app的hack。但是我安装完，没发现什么很强大包，感觉也没什么用。 


## 更新
平板一直都在吃灰，最近发现访问网站都报证书错误，可能是系统旧了(后面发现是电池馈电太久，时间不同步)，所以决定升级系统。这次完整贴出命令，因为发现我之前写的难以参考。在XDA网站发现有新的lineage17适合btv-w09,于是按照帖子里面的方法做：
1. 下载TWRP 3.3.1-1，lineage 17 系统压缩包， boot 镜像，我的是wifi-only版本。
2. 在平板关闭状态下，按住电源+音量减，振动的时候释放电源键，然后就进入fastboot状态，显示`phone unlock`
3. 刷入twrp `fastboot flash recovery twrp-xxx.img`
4. 刷入boot `fastboot flash boot boot-xxx.img`
5. 进入twrp，这里有点迷惑，网上说（包括我之前说的）按住电源+音量加总是无法进入twrp而是进入了华为的eRecovery,使用`adb boot-recovery`则进入了正常系统。最后发现在开发者模式下开启`高级重启`功能，重启的时候选择重启到`recovery`则能正常进入Twrp界面。可能是平板接着USB线。
6. 帖子上说了，升级lineage需要Wipe，彻底初始化但保留系统，执行后连sdcard目录也会格式化，剩几个标准目录。
7. 用adb push 将系统安装压缩包发送到平板的存储空间，`adb push Lineage-xxx.zip /sdcard`，然后在TWRP中选择 Install，选择这个zip 进行安装。 不过发现这个zip文件校验失败，不管他了。

然后就重启进入新版本的Lineage 17(android 10)，可惜的是没有相机功能。


## 2025年重新刷lineageos20
同样问题又出现了

1. 进入 fastboot 模式 `adb reboot bootloader`， 如今没有adb reboot fastboot命令了， fastboot模式和bootloader模式应该是一样的
2. 刷入新的recovery `fastboot flash recovery ~/Downloads/recovery.img`
3. 进入recovery，格式化data分区
4. 在recovery中点击Apply Update， 再点击Apply from ADB， 执行线刷
5. 在电脑输入`adb sideload ~/Downloads/lineage-20.0-20250102-RELEASE-btvw09.zip`

但是第5步出现错误
```
adb sideload ~/Downloads/lineage-20.0-20250102-RELEASE-btvw09.zipmain  
adb: sideload connection failed: insufficient permissions for device: missing udev rules? user is in the plugdev group
See [http://developer.android.com/tools/device.html] for more information
adb: trying pre-KitKat sideload method...
adb: pre-KitKat sideload connection failed: insufficient permissions for device: missing udev rules? user is in the plugdev group
See [http://developer.android.com/tools/device.html] for more information 
```

查看设备情况提示
```
adb devices
List of devices attached
SLYDU17401001667        no permissions (missing udev rules? user is in the plugdev group); see [http://developer.android.com/tools/device.html]
```

网上查找解决办法，尝试了好几个，最终通过
```sh
# 添加udev规则
cat /etc/udev/rules.d/51-android.rules
SUBSYSTEM=="usb", ATTR{idVendor}=="id_you_copied", MODE="0666", GROUP="plugdev"

sudo chmod a+r /etc/udev/rules.d/51-android.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```

上面还是不行，最后拔掉usb线又重新插入就正常了