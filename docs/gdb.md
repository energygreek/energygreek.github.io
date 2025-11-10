---
date: 2020-07-20
tags:
  - c
  - cpp
  - gdb
---

# gdb调试容器内的进程 

先来个[gdb手册链接](https://undo.io/resources/cppcon-2015-greg-law-give-me-15-minutes-ill-change/)  
这里记录一下自己常用的技巧

## 恢复debug符号

通常上线的程序要strip符号表来减少**内存占用**和硬盘占用，会执行以下命令
```bash
## 1. 只保留debug表的内容（可选），为以后调试用，生产时不用
objcopy --only-keep-debug foo foo.dbg

## 2. 将foo的debug表删除，不要任何调试信息，用于生产使用
objcopy --strip-debug  foo 

```

如果生产出问题了，就需要恢复debug信息，add-gnu-debuglink参数可以是foo.dbg, 也可以就是strip之前的foo文件  
所以第一步是可选的 

```
## 3. 恢复debug信息
objcopy --add-gnu-debuglink=foo.dbg foo
```

但其实第三步也是可选的，只要把foo.dbg放到foo相同位置时，gdb就可以读取到debug信息

## 使用.gdbinit

.gdbinit 文件是类似.bash_profile的文件，一般在home目录或者项目根目录, gdb 自动读取    
也可以手动读取其他位置的, 在gdb执行`source /somedir/.gdbinit`

通常可以在这个文件里定义一些设置例如`set pagination off` 关闭输出分页, 就不用每次交互确定  
还可以定义命令 如pvector可以打印vector内的所有成员, 非常比较方便

## gdb 调试容器内的进程

因为容器内运行的程序都是strip了debug符号的，所以之前调试都是先将debug文件复制到容器对应位置，然后将源码也映射到容器内，而且还要安装gdb和重新开启priveledge模式， 非常繁琐。   
所以尝试如下操作：  
1. 能不能单独一个容器安装gdb和符号和代码文件， 跨容器去调试另一个容器的进程呢？ 
2. 能不能在宿主机上直接调试容器进程呢？   

经过一番搜索，没有第一种方案的实践办法，但第二种方案亲测可行！

## 步骤

1. 先启动容器，一般业务主进程的pid为1  
2. 在容器外面执行`ps aux | grep xxx`，因为在很多情况是多个容器跑同一个程序，所以`ps aux` 通过参数名来确定进程的pid， 当然最好的方式是通过进程namespace了 
2. 使用sudo gdb -p pid， 提示缺少`Missing separate debuginfo for target`
5. 将debug符号文件复制到容器对应容器的目录
4. 再次attach，提示缺少源文件`/builds/project/src/aaa.cpp`，这是cmake编译时的目录，需要替换
3. 在gdb里执行命令 set substitute-path /builds/ /home/my-home/  # from -> to
6. 然后使用l可以查看对应的源代码了


## 总结

可以直接在宿主机上运行gdb来调试容器进程，并替换代码路径为宿主机的目录。 

ps: gdb 启动时默认会读取~/.gdbinit 文件, 但是gdb会提示, 这是由于调试docker需要sudo权限，而我的gdbinit在非root目录

```
warning: File "/home/h/.gdbinit" auto-loading has been declined by your `auto-load safe-path' set to "$debugdir:$datadir/auto-load".                                                        
To enable execution of this file add                                                                                                                                                          
        add-auto-load-safe-path /home/h/.gdbinit                                                                                                                                            
line to your configuration file "/root/.gdbinit".                                                                                                                                             
To completely disable this security protection add                                                                                                                                            
        set auto-load safe-path /                                                                                                                                                             
line to your configuration file "/root/.gdbinit".                                                                                                                                             
```

也就是需要手动执行` set auto-load safe-path / ` 或者将 `add-auto-load-safe-path /home/h/.gdbinit`写入/root/.gdbinit 

