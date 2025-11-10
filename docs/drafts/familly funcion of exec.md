---
comments: true
draft: true
date: 2025-04-11
tags: [c]
---

# exec family functions

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


  pid=fork();
  if(pid == 0)
  execle("/usr/bin/sh", "sh","-c" ,"echo $ENV1", NULL, (char*[]){"ENV1=HELLO", NULL});
}
```