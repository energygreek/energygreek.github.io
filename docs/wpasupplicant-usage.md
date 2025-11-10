---
title: wpa管理wifi接口
date:
  created: 2024-05-07 
  updated: 2025-06-14
tags:
  - raspberryPi
---
# 用 wpasupplicant 管理wifi连接
用树莓派管理wifi连接的方法有 ：
* 通过配置文件/etc/wpa_supplicant/wpa_supplicant.conf
* raspi-config
* 其它工具iwctl nmtui等

后来发现配置文件管理wifi的优先级不生效，不能选择网络，当然可以用nmtui很方便，不过要安装新包。后来发现了wpa_cli工具很方便，来自安装包wpasupplicant

## wpa_cli
先要选择网络接口，树莓派就是wlan0。
* 查看可用网络 `wpa_cli -i wlan0 list_networks`  这个命令会打印可用网络的id和名称
* 选择某个网络 wpa_cli -i wlan0 select_networks <网络id>

	wpa_cli 还有交互模式
## 参考
https://superuser.com/a/759153