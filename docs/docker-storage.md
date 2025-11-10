---
tags:
  - docker
date: 2021-08-13
---

# docker storage

docker 有3种方式使用系统存储:

- bind: 
  宿主机目录映射方式，启动容器命令添加参数`-v dir1:dir2`。将宿主机的目录dir1挂载到容器的dir2目录，实际执行的`mount -o bind`类似命令
- volume: 
  由docker管理的存储，先创建后使用，且可以在多个容器中共享，是官方推荐的方式
- tmpfs: 
  使用宿主机的内存作为存储，这种使用内存文件系统来挂载到容器目录, 使用参数`--mount type=tmpfs,destination=/dir`, tmpfs是退出即释放的一种文件系统


## docker使用的文件系统

docker使用的文件系统经过很多变化，而且在各发行版下可能不同，但目前主流的是overlay2，执行`docker info ` 查看当前使用的是overlay2
```
sudo docker info | grep  Storage                                                                                                                                              
 Storage Driver: overlay2
```

除了overlay2，还有aufs（ubuntu），devicemapper（centos），btrfs和zfs。他们的实现都不同，都能支持分层和支持写时复制（Cow/copy-on-write），而他们实现的方式有区别，所以效率也有区别

* 分层：
镜像都是分层的，在Dockerfile构建时，每次执行COPY/RUN时，都会增加一层
* 写时复制:
在容器或者DockerFile执行修改操作时，包括权限修改，会将lower layer 的文件复制到container层再修改

而容器就是在镜像顶层`压栈`了一个`可写`层，而且是临时的，当容器销毁时，这层的文件也会删除

## overlay的优势

1. page caching， 可以在多个不同实例之间共享
2. 不同层之间，相同文件使用硬连接， 节省inode 和 大小

写时复制 copy-up 会导致第一次写时造成延迟，特别是大文件，拷贝起来费时。 但第二次就不会延时， 而且overlay2 有caching， 相比其它文件系统，更减少延时

## overlay的问题

1. 实现不够完全， 例如没有实现uname 
2. 先只读打开一个文件 open（read）， 再读写打开相同文件open（write）， 两个fd 会对应2个不同文件， 第一个对应的lower的文件，第二个造成写时复制，对应容器里的文件。 
  * 规避方法是先执行touch 操作。 现实的例子是 yum 需要安装yum-plugin-ovl。 但这个只有7.2才支持， 之前的话就需要先`touch /var/lib/rpm/*`

## 最佳实践

1. 使用ssd 
2. 对于写操作比较多的场景，例如数据库，应使用映射文件(bind)或者volume。这样跳过了overlay的复杂操作，直接使用主机的文件系统

## overlay的增删改

当运行docker容器时查看挂载

```
overlay on /var/lib/docker/overlay2/04ea1faa8074e5862f40eecdba968bd9b7f222cb30e5bf6a0b9a9c48be0940f2/merged type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/B74PWZCBMRCWXFH5UL2ZXB5WEU:/var/lib/docker/overlay2/l/WNHICVPVSDNUGSCZW435TPSMOK,upperdir=/var/lib/docker/overlay2/04ea1faa8074e5862f40eecdba968bd9b7f222cb30e5bf6a0b9a9c48be0940f2/diff,workdir=/var/lib/docker/overlay2/04ea1faa8074e5862f40eecdba968bd9b7f222cb30e5bf6a0b9a9c48be0940f2/work)

```
docker 将镜像的文件挂载为只读， 将容器层挂载为可读可写。 文件系统可以分为2部分
upper（容器层） + lower （镜像层）

  * 当在容器里执行写时， 如果文件不存在， 会依次遍历lower。如果都不存在就会在upper层创建文件
  * 读也相同
  * 删除时会创建一个without 来隐藏， 这是为什么即使删除容器里的文件， 镜像还是会增大。 
  * 删除目录情况也差不多


## 特殊情况

在保存容器后(docker commit)，会多一层，里面包含了修改的文件，以及删除后生成的without文件，然后生成镜像

但对于以下特殊目录文件不会提交， 因为这些文件是运行时docker 要根据用户配置进行修改的。  

1. /etc/hostname 
2. /etc/hosts 
3. /etc/resov.conf

例如docker 的link选项，会在容器的hosts 文件里定义对应的容器名->容器ip

## 手动mount overlayfs的例子
1. 原本目录，文件都分散在不同目录ABC
```
.
├── A
│   ├── aa
│   └── a.txt
├── B
│   ├── a.txt
│   └── b.txt
├── C
│   └── c.txt
└── worker
    └── work [error opening dir]

```
2. overlay 挂载到/tmp/test目录 `sudo mount -t overlay overlay -o lowerdir=A:B,upperdir=C,workdir=worker /tmp/test/`
3. 查看test目录 
```
/tmp/test/
├── aa
├── a.txt
├── b.txt
└── c.txt
```
```
mount  | grep 'overlay'
overlay on /tmp/test type overlay (rw,relatime,lowerdir=A:B,upperdir=C,workdir=worker)
```

## 参考

https://docs.docker.com/storage/storagedriver/

