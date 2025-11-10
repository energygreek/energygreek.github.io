---
comments: true
date: 2024-12-17
tags: [linux, fork, exec]
---

# linux子进程使用方法
Linux的exec(3)是一系列函数。之前总是稀里糊涂地用，这次搞清楚了。
```c
       #include <unistd.h>

       extern char **environ;

       int execl(const char *pathname, const char *arg, ...
                       /*, (char *) NULL */);
       int execlp(const char *file, const char *arg, ...
                       /*, (char *) NULL */);
       int execle(const char *pathname, const char *arg, ...
                       /*, (char *) NULL, char *const envp[] */);
       int execv(const char *pathname, char *const argv[]);
       int execvp(const char *file, char *const argv[]);
       int execvpe(const char *file, char *const argv[], char *const envp[]);
```

## 规律
这6个函数可以分为两组'execl组'和'execv组'。 后缀p表示从$PATH中查找程序， 后缀e表示最后一个参数是数组，里面是环境变量。

## 验证
这里通过c语言和python调用命令`cat hello.txt`， 验证每个函数。

```c
#include <unistd.h>

int main(){
  int pid=-1;
  pid=fork();
  if(pid == 0)
  execl("/usr/bin/cat", "cat", "hello-execl.txt", NULL);

  pid=fork();
  if(pid == 0)
  execlp("cat", "cat", "hello-execlp.txt", NULL);

  pid=fork();
  if(pid == 0)
  execle("/usr/bin/cat", "cat", "hello-execle.txt", NULL, (char*[]){NULL});

  pid=fork();
  if(pid == 0)
  execv("/usr/bin/cat",  (char*[]){"cat", "hello-execv.txt", NULL});

  pid=fork();
  if(pid == 0)
  execvp("cat",  (char*[]){"cat", "hello-execvp.txt", NULL});

  pid=fork();
  if(pid == 0)
  execve("/usr/bin/cat",  (char*[]){"cat", "hello-execve.txt", NULL}, (char*[]){NULL});
}
```
注意当使用选项参数时，选项和参数需要分开成两个字符串。例如`grep -n 10 "target" file`命令， 调用方法是
```c
execlp("grep", "grep", "-n", "10",  "target", "file", NULL);
```

使用环境变量，要用sh来拓展ENV1符号

```c
execle("/usr/bin/sh", "sh","-c" ,"echo $ENV1", NULL, (char*[]){"ENV1=HELLO", NULL});
```

python 的os库的方法更全，且不需要带NULL
```py
>>> [method for method in dir(os) if method.startswith("exec")]
['execl', 'execle', 'execlp', 'execlpe', 'execv', 'execve', 'execvp', 'execvpe']
```

实现`cat hello.txt`
```py
import os
os.execlp("cat", "cat", "hello.txt")
```

使用环境变量, 同样要用sh来拓展ENV1符号
```py
import os

env = os.environ.copy()
env["ENV1"]="hello"
os.execlpe("sh", "sh", "-c", "echo $ENV1", env)
```

## 实用替代方法
exec族的函数都会替换当前进程的堆栈，所以需要用fork生成新的进程。更方便的方法是在c中用system, python中用subprocess