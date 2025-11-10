---
date: 2021-10-14
draft: true
tags: ['cpp','thread']
---

# 线程局部变量

众所周知的，所有线程共享堆栈，所以主线程的变量都是公用的，但如何让每个线程单独拥有一份变量呢

## 使用`pthread_key_create`

## 使用`std::thread_local`
