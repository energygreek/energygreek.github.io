---
date: 2020-08-01
tags: [python]
---


# python decoder 介绍
python decoder（装饰器）是python的高级用法，用来拓展函数或类的功能。不过也增加初学者
阅读代码的难度

例如flask可以在需要验证用户登录的请求函数上添加装饰器，只有当用户登录了才可以调用该>页面，非常方便

## 原理 

decoder 其实也是一个函数，装饰器函数会返回真实函数，并会增加额外的代码，如

```python
# 定义wraper
def log(func):
    def wrapper(*arg):
        print('func called')
        return func(*arg)

    return wrapper

# 使用
def hello(Name=None):
    print('Hello %s'%Name)

new_hello = log(hello)

new_hello('bob')

```
输出
func called
Hello bob

## 使用wraper 的原因

提供额外的功能，例如计时，日志等。 另外wraper还有更多强大的功能

```python
def fib(n):
    if n <= 1: return 1
    return fib(n-1) + fib(n-2)
```

这种递归调用非常费时，而使用wraper 可以提供一个缓存器

```python
def memory(func):
    cache = {}
    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]
    return wrapper

@memory
def fib(n):
    if n <= 1: return 1
    return fib(n-1) + fib(n-2)
 

## 给类wraper

上面是函数wraper， 类也可以作为wraper， 但需要类支持（）调用，类似c++的类函数。
在python里需要实现 `__call__` 方法

```python
import functools

class MemoryClass():
    def __init__(self, is_logging, func):
        self.is_logging = is_logging
        self.func = func
        if self.is_logging:
            print('logging')
        self.cache = {}

    def __call__(self, *args):
        if args not in self.cache:
            self.cache[args] = self.func(*args)

        return self.cache[args]

def memory(is_logging):
    return functools.partitial(MemoryClass, is_logging)


@memory(is_loging=True)
def fib(n):
    if n <= 1: return 1
    return fib(n-1) + fib(n-2)


print(fib(5))
```
类wrapr 可以增加参数，相比函数装饰器，类装饰器具有更加灵活，封装性更好等优点~


## wraper 带来的问题


1. 改变原来的函数属性，例如函数名， 因为装饰器用新函数替换了原来的旧函数。 所以新函>数少了旧函数的属性
解决办法

```python
import functools

def advertising(func):
    @functools.wraps(func)
    def wrapper(*args):
        print('欢迎关注微信公众号: Charles的皮卡丘')
        return func(*args)
    return wrapper

@advertising
def add_1(a, b):
    '''加法运算'''
    return a + b

```

2. 获取参数

```python

import inspect
import functools

def check(func):
    a = 0
    @functools.wraps(func)
    def wraper(*args, **kwargs):
        a += 1
        print(a)
        # 在wraper里检查参数
        getcallargs = inspect.getcallargs(func, *args, **kwargs)
        if getcallargs.get('divisor') <= 1e-6 and getcallargs.get('divisor') >= -1e-6:
            return False
        return func(*args, **kwargs)

    return wraper


@check
def dowork(arg1, arg2):
    return arg1 + arg2

dowork(1,2)

```

此时会报错，a在赋值之前引用了。 需要将a声明为 `nonlocal`

