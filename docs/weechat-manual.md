---
date: 2021-01-19
tags: [tool,irc]
---

# weechat 使用方法

weechat 是一个irc客户端， 在终端中运行，不需要gui桌面，非常方便。 这里记录配置方法

主要参考[链接](https://samirparikh.com/blog/setting-up-weechat-commandline-irc-client.html)

## 安装和运行weechat

安装后，直接在终端运行`weechat`，就能进入weechat

## 配置

### 添加server 

irc有很多server, 常用的是freenode。 另外有darkscience。 添加方法如下，另外weechat里面命令都是以'/' 开头，还能使用tab补全
```
/server add freenode chat.freenode.net/6697 -ssl
```
irc 除了有很多服务器url, 每个url 还有多个端口供链接，比如6697

### 设置昵称

昵称默认是linux系统的用户名，昵称用来在聊天中显示名称
```
/nick mynickname
```
**注意：
手动修改昵称不是简单的事情，特别是当已经连接serer时。 比较方便的办法就是直接改配置文件
```
# ～/.weechat/irc.conf
nicks = "malloy,malloy1,malloy2"
```
带后缀的昵称用来当malloy占用时的备用，比如网络重连时昵称有冲突就需要备用昵称

### 注册昵称

因为有很多聊天室要求注册后才能进入，所以先要使用邮箱注册
```
/msg nickserv register userpassword example@email
```

然后irc 会提示查收邮件，在邮件里面提示你在irc输入命令完成注册

### 加入聊天室

聊天室都是以#开头，例如我使用的 '#archlinuxcn'
```
/join #weechat
```

### 退出聊天室

```
/parted "leave message"
```
或者
```
/close
```

## 自动设置

设置打开weechat时，自动登陆和自动进入聊天室

```
/set irc.server.freenode.autoconnect on
/set irc.server.networkname.autojoin "#channel1,#channel2"
```

## 界面设置

先介绍weechat的各名称对应命令 : 
聊天室称为buffer, 切换聊天室命令 `/buffer n`  
最左边的聊天列表是bufferlist bar  
进入buffer后，最右边的是nicklist bar，一个聊天室界面叫做window， 所以下面的命令设置就与 /bar /windows 有关

隐藏和显示bufferlist
```
/bar hiden bufferlist 
/bar show bufferlist
/bar toggle bufferlist
```
分屏显示所有聊天室，水平分屏splith,竖直分屏 splitev
```
/window splitv 
```
窗口切换
```
/window +1
/window -1
```
窗口合并
```
/window merge
```

## 安全设置

1. 上面添加服务器时，已经配置了ssl选项，就表示使用通讯加密了，但irc的聊天记录一般是对外公开的  
2. 设置weechat开启密码， 进入weechat前验密
```
/secure passphrase superSecretPassphrase
```

## 自动化设置和鉴权
注册好昵称之后，配置自动化登陆，而不需要手动执行命令

设置irc登陆昵称，作为identify的参数，务必要跟register的保持一致
```
/secure set networkname_nickname password
```
设置irc登陆密码，作为identify的参数，务必要跟register的密码保持一致
```
/secure set networkname_password password
```

设置weechat启动时自动登陆服务器，以及自动加入channel
```
/set irc.server.libera.autoconnect on
/set irc.server.libera.autojoin "#archlinux-cn,#c,#c++"
```
以及自动验密
```
/set irc.server.networkname.command "/msg nickserv identify ${sec.data.networkname_nickname} ${sec.data.network_password}"
```
其实显而易见地，`sec.data.networkname_nickname`和`sec.data.network_password` 都只是保存在sec.data里的变量，其实可以两个变量合成一条也没问题

## 更多功能

weechat有很多拓展插件，可以完成很多事情，比如自动回复，远程控制，连接telegram等, [参考](https://weechat.org/scripts/)
