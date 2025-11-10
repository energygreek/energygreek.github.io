---
date: 2021-01-28
tags: [linux,systemd]
---

# 创建 systemd 服务

说一个老话， 现在systemd作为linux的启动管理和服务管理已经越来越重要了， 上周考试也遇到用systemd 来管理容器，这里记录一下如何编写systemd服务

## 关于systemd

systemd是只能运行在Linux上的init, 也就是启动后看到的1号进程。 除了启动， systemd还管理着很多东西，例如网络（systemd-networkd）， 域名解析（systemd-resolved），为服务创建socket(systemd.socket) 文件系统挂载，还有系统和用户的服务  
systemd太大，说不完，需要查看各种文档

## systemd 的两种使用模式

systemd 分为system级别和user级别， 对应的unit文件分别在/etc/systemd/ 和 ～/.config/systemd/下， 前者是系统级别，后者是用户级别。 用户只能运行自己设置的服务

```
systemctl start system_service.service
```
而普通用户只能执行
```
systemctl --user user_service.service
```

这个name就是文件名称，例如必须'/etc/systemd/system/'下存在'system_service.service'文件，在能执行第一条的命令、 必须在 '~/.config/systemd/user/'下存在'user_service.service'在能执行第二条命令

### 系统服务以其他用户运行服务
系统级别的服务默认会以root来运行服务，但是也可以设置以其他用户来运行来最小化权限，例如音视频服务。也可以以某个用户来执行，那么service unit 文件就变为'system_service@user.service'

```
# system_service@user.service
[Unit]
Description=Watchman for user %i
After=remote-fs.target
Conflicts=shutdown.target

[Service]
ExecStart=/usr/local/bin/watchman --foreground --inetd
ExecStop=pkill -u %i -x watchman
Restart=on-failure
User=%i
Group=users
StandardInput=socket
StandardOutput=syslog
SyslogIdentifier=watchman-%i

[Install]
WantedBy=multi-user.target
```

上面的服务以下面的socket 单元启动， 前提要这个服务实现接收socket,通过`sd_listen_fds(3)`
```
# system_service@user.socket

[Unit]
Description=Watchman socket for user %i

[Socket]
ListenStream=/var/facebook/watchman/%i-state/sock
Accept=false
SocketMode=0664
SocketUser=%i
SocketGroup=othergroup

[Install]
WantedBy=sockets.target
```

### 普通用户运行服务

注意， 普通用户因为只会以自己的身份启动，所以不能想系统服务那样指定'User/Group'
```
[Unit]
Description=tun2socks for vpn
#Requires=ssh_to_alpha.service

[Service]
Type=simple
ExecStart=/usr/bin/badvpn-tun2socks 

[Install]
WantedBy=default.target

```

若希望这个用户自定义服务能自启动， WantedBy需要设置成'default.target'


## 自动创建unit-file

以下命令可以自动在对应目录创建*.service文件
```
systemctl --user --force --full edit test.service 
```






