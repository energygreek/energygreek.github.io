---
date: 2023-09-08
tags: [debian,tool]
---

# 随身wifi工具刷机
本文记录了淘宝卖的随身wifi刷机成Debian，还能收发短信。还有救砖的艰辛过程，不过结局美满。

## 依赖
* edl备份恢复工具，https://github.com/bkerler/edl, 下载源代码后执行`git submodule update --init --recursive`和 `pip install -r requirements.txt`
* openstick编译好的debian https://github.com/OpenStick/OpenStick。 在release页面下载 base.zip, debian.zip, firmware-ufi001c.zip，boot-ufi001c.img
* apt 安装 adb fastboot

## 切卡
刚接触时看网上的都是在后台页面切卡，但是刷机了没有后台页面如何切卡呢？ 为了切卡又重刷安卓，导致变砖了。参考网上资料发现，Debian下可以直接切。
```
debian默认卡槽位1:

echo 1 > /sys/class/leds/sim:sel/brightness
echo 0 > /sys/class/leds/sim:en/brightness
echo 0 > /sys/class/leds/sim:sel2/brightness
echo 0 > /sys/class/leds/sim:en2/brightnessR
modprobe -r qcom-q6v5-mss
modprobe qcom-q6v5-mss
systemctl restart rmtfs
systemctl restart dbus-org.freedesktop.ModemManager1.service

esim槽位3:

echo 0 > /sys/class/leds/sim:sel/brightness
echo 0 > /sys/class/leds/sim:en/brightness
echo 1 > /sys/class/leds/sim:sel2/brightness
echo 0 > /sys/class/leds/sim:en2/brightness
modprobe -r qcom-q6v5-mss
modprobe qcom-q6v5-mss
systemctl restart rmtfs
systemctl restart dbus-org.freedesktop.ModemManager1.service

其他两个槽位

槽位2:

echo 0 > /sys/class/leds/sim:sel/brightness
echo 1 > /sys/class/leds/sim:en/brightness
echo 0 > /sys/class/leds/sim:sel2/brightness
echo 0 > /sys/class/leds/sim:en2/brightness
modprobe -r qcom-q6v5-mss
modprobe qcom-q6v5-mss
systemctl restart rmtfs
systemctl restart dbus-org.freedesktop.ModemManager1.service

槽位4:
echo 0 > /sys/class/leds/sim:sel/brightness
echo 0 > /sys/class/leds/sim:en/brightness
echo 0 > /sys/class/leds/sim:sel2/brightness
echo 1 > /sys/class/leds/sim:en2/brightness
modprobe -r qcom-q6v5-mss
modprobe qcom-q6v5-mss
systemctl restart rmtfs
systemctl restart dbus-org.freedesktop.ModemManager1.service

有些棒子esim槽位不一样槽位3不行自行试试槽位2 4
```
https://github.com/OpenStick/OpenStick/issues/49#issuecomment-1568202001

## 刷机
先使用edl全量备份，方法在最下面。先刷base包，解压base.zip后执行里面的`flash.sh`。然后刷debian包，解压debian.zip后执行里面的`flash.sh`。这时进入系统发现可能出现问题（我也没明显感觉到问题），因为他们是ufi001b的。
所以再刷适配ufi001c的包，解压firmware-ufi001c.zip。
```
cd firmware-ufi001c
adb push ./* /lib/firmware
adb reboot bootloader
fastboot flash boot boot-ufi001c.img
fastboot reboot
```
但是在刷ufi001c的包之前我能找到modem网络，刷完就没了，所以有了下面基带恢复的步骤。

## 基带
基带文件在随身wifi的modem分区，先用edl备份modem分区。当然下面介绍了edl全量备份，跳过了userdata，估计是root和home目录。

### 恢复基带文件
我在安装debian系统时，发现移动网络不可用，于是参考网上说的，将原始基带恢复后正常。 基带文件名为"modem.bin", edl全量备份后在dumps目录下。直接mount该文件，然后将modem.*和mba.mbn取出来，当发现刷机后发现移动网络不能使用时，重新拷回去就能用了。
```
sudo mount modem.bin /mnt
```
复制到firmware-ufi001c目录，覆盖之前的文件。
```
cp /mnt/modem.* .
cp /mnt/mba.mbn .
```

再次拷贝到wifi棒子的/lib/firmware目录, 配合再刷一次适配001C的boot文件。
```
adb push ./* /lib/firmware
adb reboot bootloader
fastboot flash boot boot-ufi001c.img
fastboot reboot #重启
```

然后就能找到modem网卡了。

## 重启modem

有时重刷了firmware，还是出现没有modem网卡的情况，也就是`mmcli -m 0`提示找不到modem设备的时候，需要执行下面的命令。或者直接等一会重启ModemManager
```
systemctl stop ModemManager
qmicli -d /dev/wwan0qmi0 --uim-sim-power-off=1 && qmicli -d /dev/wwan0qmi0 --uim-sim-power-on=1
systemctl start ModemManager
```

## 短信收发和转发邮件
下载两个python文件，地址 https://gitee.com/jiu-xiao/ufi-message。配置smtp.py中的邮箱信息，我使用的QQ邮箱登录，然后转发到另一个邮箱。开始发送不成功，修改后能发送了，内容如下。
```
#!python3

my_sender='@qq.com'
my_user='@hotmail.com'
server_address='smtp.qq.com'
server_port=465
server_passwd='' #填写qq邮箱授权码


import smtplib
from email.mime.text import MIMEText
from email.utils import formataddr
import datetime
import time

def mail(text):
    msg=MIMEText(text,'plain','utf-8')
    msg['From']=formataddr(["随身Wifi",my_sender])
    msg['To']=formataddr([my_user,my_user])
    msg['Subject']="转发 "+time.strftime("%H:%%M:%S %m/%d")

    server=smtplib.SMTP_SSL(server_address,server_port)

    server.login(my_sender,server_passwd)
    server.auth_plain()
    server.sendmail(my_sender,[my_user,],msg.as_string())
    server.quit()
```

然后就能写进crontab自动跑了。

## 救砖
### edl使用
github上的开源工具，可以替代Windows那些高通工具，Miko，星海啥的。是我这Linux用户的福音，并且简单，windows常常失败。
使用edl工具之前要先重启到edl也就是9008模式，使用命令`adb reboot edl`，不需要按RST键。

备份：推荐第一种方式，第二种方式生成的一个文件刷到另一个设备失败，但是可以刷MiKo导出的单文件。
```
edl rl dumps --skip=userdata --genxml
or
edl rf flash.bin #单文件
```

恢复:
```
edl qfil rawprogram0.xml patch0.xml .
```

### 恢复分区
刷了debian后，分区会变，这样导致有些分区不存在所以无法刷回去。这里介绍下恢复分区的方法。不过上面的edl恢复应该不需要这个操作。

备份分区
```
./edl gpt . --genxml
```
备份后有两个文件，恢复时只用gpt_main0.bin文件，恢复分区命令
```
./edl w gpt gpt_main0.bin
```
打印分区来验证下

```
./edl printgpt
```

## 参考
https://forum.openwrt.org/t/uf896-qualcomm-msm8916-lte-router-384mib-ram-2-4gib-flash-android-openwrt/131712/160
https://www.kancloud.cn/handsomehacker/openstick/2636505
