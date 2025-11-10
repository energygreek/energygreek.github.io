---
date: 2020-08-10
tags: [docker]
---

# Dockerfile 中ENTRYPOINT和CMD的区别

Dockerfile 中，ENTRYPOINT和CMD来都可以用来设置默认命令，而代替`docker run`时的命令，也可以一起使用  

单独使用ENTRYPOINT或CMD时，其区别在于对`docker run`的参数处理:  
CMD的命令会被`docker run`的参数覆盖，而ENTRYPOINT是追加

例如：
```
# image1
FROM ubuntu
ENTRYPOINT ["echo", "hello world"]
```
```
# image2
FROM ubuntu
CMD ["echo", "hello world"]
```

执行结果
```
# docker run image1
hello world

# docker run image1 hostname
0c33ff9af81c
```

```
# docker run image2
hello world

# docker run image2 hostname
hello world 0c33ff9af81c
```


## 实际情况多是entrypoint与cmd组合使用

cmd 为entrypoint 提供默认参数， 即如果'docker run'给了参数时，覆盖CMD参数，否则使用CMD提供的参数

```
FROM ubuntu
ENTRYPOINT ["top", "-b"]
CMD ["-c"]
```

## 首先说cmd和entrypoint 都有两种模式

1. shell模式， 即没有用数组来传递参数, 直接接命令如 CMD top -p 或者 entrypoint top -p 
2. exec 模式， 即使用数组表示如 CMD ["top", "-p"]， 注意必须用双引号，因为这个是用json来传递的

两者区别:  
shell模式不会接受命令行参数，而且1号进程是sh，而非如上的top。 这样的话top不能接受到`docker stop container`时的信号。 
而一般我们写的程序都是捕捉SIGTERM来进行peaceful exit，那么就一定不能使用shell模式

exec模式时， 可以接受docker run image param， 这个param会传递给top作为参数。 而且top 是1号进程，能接收信号。

细微区别： 由于shell模式是先启动sh， 符号解析由shell执行，例如`$HOME`的展开由shell进行， 而用exec模式时，由docker 进行

总之，推荐使用exec 模式


## 额外话, 减小镜像体积

使用'\' 或者 '&&' 将命令连起来可以减小镜像层数，从而减小体积。另外build时，单独一行的命令会在新的layer 层执行。有些命令产生的cache，会持续影响到下一层  
例如`RUN apt-get dist-upgrade -y`，如果不想保留cache，需要在build时加入参数`docker build --no-cache.` 

还有最有效减小镜像体积的方式，也常见的是最后用空白镜像，'FROM SCRATCH', 这样把其他的层都干掉了, 只保留一层镜像
