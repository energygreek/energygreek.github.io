---
title: bash learning
date: 2021-05-21
tags: [bash]
---


# bash使用技巧收集

Linux的命令行是非常强大的生产工具，现实中的问题好多可以不写复杂的编程代码而通过命令来解决, 而且不过几行代码而已  

## shell 内建命令

当执行which的时候，比如`which echo` 会输出'shell built-in command'而不是返回路径，就说明这个命令是内建命令  
shell在执行内建命令时是直接执行，而非内建命令则fork一个子进程来执行

## 命令行的问题很多与缓冲buff、重定向pipe、delimiter分割符有关



## 技巧, 用命令代替编程

### 使用xargs创建多进程并发

xargs可以将输入多进程并发处理，配合`find` 非常厉害。 配合wget就是一个并发爬虫了

```
EXAMPLES
       find /tmp -name core -type f -print | xargs /bin/rm -f

       Find files named core in or below the directory /tmp and delete them.  Note that this will work incorrectly if there are any filenames containing newlines or spaces.

       find /tmp -name core -type f -print0 | xargs -0 /bin/rm -f

       Find files named core in or below the directory /tmp and delete them, processing filenames in such a way that file or directory names containing spaces or newlines  are  correctly
       handled.

       find /tmp -depth -name core -type f -delete

       Find  files  named  core in or below the directory /tmp and delete them, but more efficiently than in the previous example (because we avoid the need to use fork(2) and exec(2) to
       launch rm and we don't need the extra xargs process).

       cut -d: -f1 < /etc/passwd | sort | xargs echo

       Generates a compact listing of all the users on the system.

```

## find -exec 用法

```
find . -exec grep chrome {} \;
find . -exec grep chrome {} +
```
1. \; 是转义;
2. {} 会替换成 find出的文件名。
2. ; 和 + 的区别是： 当;时，grep命令会在每个文件名上执行一次，grep执行多次， 但+时，所有的文件名会将所有的文件名一次进行grep，grep执行一次。

### 使用awk ORS 分割结果

ls 本来是行分隔的
awk将ls的换行符变为|分割
```bash
ls | awk '{ ORS="|"; print; }'
```
而`echo $(ls)` 则把ls的换行符变为空格

### 使用declare 声明变量的类型和属性

declare可以指定变量的类型'-i'为整形，'-r'制度(等同shell的readonly)，'-g'全局变量
shell默认是字符串，使用'-i'后，数学运算不需要`let`

```bash
#retry=0
declare -i retry=0
while [ $retry -lt 30 ] ; do
ps aux --cols=1024 | grep xxx
if [ $? -eq 0 ] ; then
        exit 0
fi
sleep 1
#let retry=$retry+1
retry=$retry+1
done
```
