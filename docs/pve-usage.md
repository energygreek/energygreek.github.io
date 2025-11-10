---
date: 2023-08-12 17:39:12
tags: [pve]
---

# pve入门

## lxc 配置
pve使用lxc来运行容器而不是docker。 docker对应的“镜像”概念在pve里叫`模板(template)`，默认的pve容器模板是从官网下载，非常慢，可以替换成清华源。另外在模板列表里面的模板都是比较新的，老版本可以上[清华源](https://mirrors.tuna.tsinghua.edu.cn/proxmox/images/system/)上找，例如centos7，然后通过url来上传，怎样都非常方便。

### 容器网络问题
有的容器模板没有安装红帽的NetworkManger，这导致创建容器时选择的DHCP网卡配置不会生效，即使给网卡设置静态ip也不行。有个解决办法是启动后再加一个静态ip的网卡，这样容器就有网络访问权限了，然后赶紧安装NetworkManger吧。

### 低版本容器启动不能进入console的问题
如果systemd的版本低于232，启动容器时pve提示:WARN: old systemd (< v232) detected, container won't run in a pure cgroupv2 environment!  
也打不开控制台，只能通过在宿主机上执行 `pct enter id`来进入容器。网上有帖子说在容器里面执行软件更新后，可以恢复，也许是systemd版本更新了，但是这个方法不适用于centos7。最后只能通过修改宿主机的grub，修改启动参数:
```
#/etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="systemd.unified_cgroup_hierarchy=0 quiet"
```
再执行`update-grub`命令，最后重启宿主机后问题解决。

### 低版本容器问题
pve的容器ENTERPOINT都是`/sbin/init`，跟我现在公司的管理docker方式相同。 会启动systemd，包括sshd等，对于远程访问非常方便。但是版本低的容器启动时不仅无法进入控制台，而且不会启动systemd。手动使用systemctl启动sshd时会失败，因为dbus没有启动，而启动dbus也失败。总之新版本容器就不会有这个问题，总之要配置启动参数。

对于通过/sbin/init来启动sshd的方式值得讨论，因为通过在Dockerfile中设置`ENTRYPOINT service ssh restart && bash`也可以有相同的效果。而公司的大量容器都启动systemd时，会启动udev服务，这时硬件故障，直接导致宿主机卡死。

### map directory to lxc container
pct set 110 -mp0 /srv/music,mp=/srv/music

## 虚拟机配置声音
我安装过一个win7的虚拟机作为测试环境用，而且需要使用音频，在添加音频网卡时，使用了SPICE驱动，会导致虚拟机无法启动，必须将显示器的驱动也改成SPICE。然而我使用remmina来远程访问这个虚拟机，听不到声音。 最后将音频驱动改为None，然后remmina的advance标签卡的`Auido output mode`设置`Local`就能听到声音了，虽然声音质量不佳，不过勉强能用!


## 参考
- https://forum.proxmox.com/threads/solved-warn-old-systemd-v232-detected-container-wont-run-in-a-pure-cgroupv2-environment.114736/
- https://stackoverflow.com/questions/25135897/how-to-automatically-start-a-service-when-running-a-docker-container/32179054#32179054
