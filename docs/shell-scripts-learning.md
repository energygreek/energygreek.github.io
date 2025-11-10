---
date: 2021-03-31
tags: [shell]
---

# shell 笔记
shell 是unix-like系统下，用户与系统交互的媒介，用来解析用户的输入并调用系统函数。 
而shell的实现有常见的bash,zsh,ksh等, 他们实现有很多差别，但bash最为通用

## bash 模式拓展

## bash 字符串操作
## bash 数组操作


## 环境变量

测试程序定时获取和打印环境变量
```c
int main() {
  while (1) {
    char *env = getenv("TEST_ENV");
    printf("env: %s\n", env);
    sleep(5);
  }
}
```

通过bash来修改环境变量
```sh
#test.sh
export TEST_ENV=TEST
./a.out
export TEST_ENV=NNN
```

执行`test.sh`，c程序没有更新环境变量, 所以环境变量不会变化。

```
env: TEST
env: TEST
env: TEST
```

```
#include <stdio.h>

extern char **environ;

int main() {
  char **var;
  for (var = environ; *var != NULL; ++var) {
    printf("%s\n", *var);
  }
}
```

## set unset
