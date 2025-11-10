---
title: 输出缓冲和stdbuf
date: 2021-08-02
tags: [stdbuf,buffer,linux]
---

# 起因

遇到过一个面试题, 以下程序打印几个'A'
```c
#include <stdio.h>
#include <unistd.h>

int main() {
  printf("A");
  fork();
  printf("A");
}
```
答案是4个,而以下是3个
```c
#include <stdio.h>
#include <unistd.h>

int main() {
  printf("A\n");
  fork();
  printf("A");
}
```
再把上面的编译后的可执行文件重定向到文件, 则有4个'A'
```c
#./a.out > r.log
#cat r.log
```
原因就是缓冲 

## Linux 缓冲介绍

分为无缓冲、行缓冲和全缓冲

## 演示
发现有些程序可以输出到屏幕，重定向到文件后，执行`ctl+c`退出后却没有任何内容，例如以下

```
#include <stdio.h>
#include <unistd.h>

int main(void) {
  while (1) {
    printf("Hello World\n");
    sleep(1);
  }
}
```
这个程序是死循环所以需要手动结束，且一直打印helloword。但如果将输出重定向到文件，

```
#./a.out > r.log
```
当手动退出时，文件依旧是空的
