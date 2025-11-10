---
comments: true
draft: true
date: 2025-04-11
tags: [linux,c]
---

# isatty 打印日志
程序运行时往文件里写日志时页缓冲，一旦程序奔溃前没有flush就会导致日志截断，但是每次写日志都flush就不如无缓冲的write了。 而往命令行打印日志则是行缓冲，但是会影响使用，如果用nohup来启动，日志还是打印到nohup中了，也是存在截断问题。

通过isatty来判断程序是写往控制台还是文件， 如果不是控制台就往文件里写，并加上flush

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    // Check if stdin (file descriptor 0) is a terminal
    if (isatty(STDIN_FILENO)) {
        printf("Input is coming from a terminal.\n");
    } else {
        printf("Input is not coming from a terminal (e.g., pipe or file).\n");
    }

    // Check if stdout (file descriptor 1) is a terminal
    if (isatty(STDOUT_FILENO)) {
        printf("Output is going to a terminal.\n");
    } else {
        printf("Output is not going to a terminal (e.g., redirected to a file).\n");
    }

    return 0;
}
```