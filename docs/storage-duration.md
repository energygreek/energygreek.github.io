---
tags:
  - c,
  - cpp
date: 2021-04-29
---

# c++ 存储周期、链接和作用域
## c++中变量和函数的三个重要属性

存储周期类型： 有关变量的创建和销毁
链接类型： 有关变量函数的内存位置
作用域: 有关变量函数的可见范围

本文讨论的标识符，包括变量和函数

## 存储说明符

`存储说明符`控制变量何时分配和释放，有以下几种

- automatic
- thread_local
- static 
- register
- mutable
- extern

说明
- automatic: 最常见的局部变量，且没有声明为static或者thread_local，位于栈上, 随着代码块的执行和结束而自动分配和销毁
- static: 静态变量, 在程序启动和结束时创建和销毁，但初始化是在第一次执行初始化代码时执行
- thread: 在线程开始和结束时分配和销毁
- dynamic: 最常见的堆上的变量, 需要执行new和delete, 

auto 在c++11中不是声明存储周期，而是类型推导符, 但这种存储周期类型的依然存在（局部变量）  

## 初始化的时机

- automatic: 必须手动初始化，换句话说局部变量必须初始化，否则值为不确定
- static: 在执行时初始化，且初始化一次，特殊情况下在执行前初始化
- thread: 因为thread_local变量自带static性质，所以认为其同于static
- dynamic: 在new时初始化

## Linkage

标识符（变量&函数）用一块内存里的值或者函数体来表示的， 而linkage决定其他相同的标识符是否指向同一块内存。c/c++有3种linkage, no-linkage, internal linkage和external linkage

* no linkage 局部变量没有linkage, 所以两个a是独立的，后面的a会覆盖前面的a，不相干。此时linkage与可见域(scope)类似
* internal linkage 表示只能在文件内部访问(file scope)，换句话就是不会暴露给链接器， 用修饰符`static`声明internal linkage，所以允许在不同文件声明两个名称&类型相同的internal linkage 标识符，他们指向不同的内存单元。 
* external linkage 表示可以在程序所有地方访问，包括外部文件(global scope)，所以是真“全局”（scope&linkage）， 所有标识符指向独一份内存。
  
### 修饰符
- 全局const变量和全局constexpr变量默认具备internal linkage, 再加上`static`没有影响
- 全局非const变量默认是external linkage， 故再加上extern没有影响。在其他文件使用`extern`声明这个变量，就能使用指向同一内存的变量
- 函数默认external linkage，故再加上extern没有影响。 在其他文件使用`extern`声明这个函数（可省），就能使用指向同一内存的函数
- 使用extern修饰全局const变量和constexpr变量可以使起具备external linkage

可见`static`和`extern`即表示存储周期，又表示linkage， static相对简单，extern则比较复杂，如以下情况
```
int g_x = 1; // 定义有初始化的全局变量（可加可不加extern）
int g_x; // 定义没有初始化的全局变量（不可加extern），可选初始化
extern int g_x; // 前置声明一个全局变量，不可初始化

extern const int g_y { 1 }; // 定义全局常量，const必须初始化
extern const int g_y; // 前置声明全局常量，不可初始化
```

所以若是定义未初始化的全局变量，不能加extern，不然就成了前置声明了。

### constexpr 特殊情况

虽然通过给constexpr添加`extern`修饰符来让其具备external属性，但不能在其他文件前置声明。因为constexpr是在编译期替换的，编译器（compile)的可见域限定在文件内，所以编译期无法知道constexpr的值，所以在编译期无法获取到其内存单元的值， 也就无法在其他文件进行声明，只能定义。

## file scope和global scope

局部变量的scope、no-linkage以及duration相同，从`{`开始到`}`结束。 理论上global scope涵盖了file scope。而linkage来规定其是否能在其他文件里使用。

## local class

local class 不允许有static data member

## 参考

https://en.cppreference.com/w/cpp/language/storage_duration

