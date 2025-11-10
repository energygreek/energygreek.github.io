---
date: 2021-10-14
tags: [docker,namespace]
---

# 容器的命名空间

容器是利用'cgroup'和'namespace' 实现软件级的虚拟化，产品有lxc和docker。不同与全虚拟化和半虚拟化，这些是硬件辅助的半虚拟化Zen(Zen也有全虚拟化)和全虚拟化kvm   
因为容器使用主机内核，不存在特权指令翻译，所以从性能上比较，容器是最高效的，其次是kvm, Zen

## cgroup和namespace

这两个是容器技术的根基，包括Docker, lxc。实际上docker开始用的lxc，后来用go重新写了libcontainer来实现控制这两个东西  
简单来说，cgroup实现对进程组的资源的限制和监控，而namespace如同c++里面类似，实现了进程组之间的隔离

但除此之外, 虚拟化系统时则还需要rootfs支持

## LXC和Docker

lxc代码合入了内核，其原理和Docker非常类似, 不过Docker有比较丰富的全套工具和方便的api   

## nsenter命令的使用

容器技术都使用到了namespace, 而nsenter可以操作namespace, 进入容器内部，以下都一docker容器为例  

使用命令, 获取到容器的首进程pid
```
docker inspect --format '{{.State.Pid}}' containername
3015
```

然后可以使用nsenter来进入容器，如同`docker exec` 一样
```
$ pwd
/home/user
$ sudo nsenter -t 3015 -m  pwd
/
```

`-m`表示挂载命名空间，这样就能访问docker里的进程的文件系统，所以能正确打印出容器进程的当前目录  
`-t`表示目标进程，nsenter可以获取以下命名空间。 跟通过执行`ls /proc/self/ns`所看到的差不多，这里self是命令本身pid
```
/proc/pid/ns/mnt    the mount namespace
/proc/pid/ns/uts    the UTS namespace
/proc/pid/ns/ipc    the IPC namespace
/proc/pid/ns/net    the network namespace
/proc/pid/ns/pid    the PID namespace
/proc/pid/ns/user   the user namespace
/proc/pid/ns/cgroup the cgroup namespace
/proc/pid/ns/time   the time namespace
/proc/pid/root      the root directory
/proc/pid/cwd       the working directory respectively
```
上面的每项都有对应的参数，而'-a'表示进入上面的所有命名空间

