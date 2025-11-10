---
date: 2021-03-02
tags: [cpp]
---

# 右值, 临时变量与引用

我们说的临时变量通常是指在语句块里定义的短暂生命周期的局部变量, 其存储周期为'auto'，但这里讨论的是c++中'temporary object', 是在老版本c++中
而语句块里定义的变量以及非引用类型参数，并不属于这里的临时变量  

c++中的临时变量出现情况：
* litteral常量, 如1
* 类型转换   // 赋值语句结束后，自动销毁
* 函数返回值 // 赋值语句结束后，自动销毁
* 表达式的值 

而临时变量的定义就是`编译器自动创建和销毁的没有名字的变量`, 也无法取地址。并且规定：  
- 常量类型的引用（`reference to const`）可以绑定到临时变量
- 非常量类型的引用（`reference to non-const`）类型不能绑定到临时变量

这个规定的原因是： 因为上面说的`编译器自动创建和销毁`，所以去修改一个会销毁变量是`没有意义的`

## 什么时候编译器要需要创建临时变量

| 创建原因| 销毁时机|
| --- | --- |
|Result of expression evaluation |	All temporaries created as a result of expression evaluation are destroyed at the end of the expression statement (that is, at the semicolon), or at the end of the controlling expressions for for, if, while, do, and switch statements. |
| Initializing const references  |	If an initializer is not an l-value of the same type as the reference being initialized, a temporary of the underlying object type is created and initialized with the initialization expression. This temporary object is destroyed immediately after the reference object to which it is bound is destroyed. |

主要是以下两个地方会需要临时变量：
* 函数接受引用类型参数的时候
* 函数返回的时候
* 类型转换的时候

举例：

函数参数
```c++
void foo(const int &arg){}
foo(1);  // 此时创建了临时变量，可以想象成'foo(int _tmp(i));' 而且这个_tmp 不能被修改
```

函数返回值
```c++
int foo(){return 1;}
int i=foo(); // 此时创建了临时变量， int _tmp = foo(); int i = _tmp;
```

类型转换
```c++
int i = 1;
double d = i; // double _tmp = i; double d = _tmp;
```

## 单独讨论这些似乎意义不大， 但是在实际编写代码的时候，可能会有疑惑

例如如何解释
```
const int& cr = 1; // ok
int &r = 1; // ng
```
这里赋值和函数传参一样， 因为1是常数，这里为常量`1`创建临时变量， 而只有`reference to const` 才能绑定到临时变量上

还有解释一个经常被引用的例子
```c++
void foo(int &arg){}
int i = 1; foo(i); // ok
double d = 1.0; foo(d); // ng
```

```c++
void foo(const int &arg){}
int i = 1; foo(i); // ok
double d = 1.0; foo(d); // ok
```
这里虽然i和d都不是常量（不会创建临时变量）， 但是因为类型不同， 发生了类型转换，创建了临时变量。同样地，只有`reference to const` 才能绑定到临时变量上
