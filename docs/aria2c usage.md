---
draft: false
comments: true
date: 2025-09-29
tags: [aria2]
---

# aria2c usage
aria2c支持很多类型的协议下载，http, bt等，支持任务列表，支持前台运行和后台运行， 还支持并发下载（单连接限速）和断点续传（前提是服务器支持）。因为现代浏览器普遍也支持断点续传和并发下载，所以aria2更适合用来批量下载。

## 前台运行
就类似wget fcurl
* 单个文件下载 aria2c "下载链接"
* 文件列表下载 aria2c -i list.txt 

## 后台运行
aria2c 本身不支持类似`-d/--daemon`参数，所以要么在终端执行下面的命令，要么用systed来运行。
* 后台运行 aria2c -c -i list.txt -j4 --load-cookies=rapidshare > /tmp/aria2c.log 2>&1 &
* 查看日志 tail -f /var/log/aria2c.log

### systemd 和 web/rpc 控制

创建文件~/.config/systemd/user/aria2cd.service，内容如下。这时只能通过rpc协议和aria2c交互，可以用curl和web来发送rpc。
```
[Unit]
Description=Aria2 Daemon

[Service]
Type=simple
ExecStart=/usr/bin/aria2c --conf-path=/etc/aria2/aria2.conf

[Install]
WantedBy=default.target
```
我认为最好的纯静态前端[YAAW for Chrome](https://github.com/acgotaku/YAAW-for-Chrome)

## 配置代理
有时aria2c运行在没有网络的服务器上，需要代理运行，如下添加http代理
```c
all-proxy=http://ip:port
http-proxy=http://ip:port
https-proxy=http://ip:port
```

## 参考
https://www.linuxquestions.org/questions/linux-software-2/putting-aria2c-in-background-769997/
