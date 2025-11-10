---
date: 2021-12-06
tags: [c]
---

# c语言中pointer和array的区别

在stackoverflow上搜到一个有趣[帖子](https://stackoverflow.com/questions/21972465/difference-between-dereferencing-pointer-and-accessing-array-elements)  
讨论了指针解引用和数组下标访问的区别，而根本原因就是`指针有时不同于数组`，这是《c专家编程》有仔细讨论的，故再看了一遍

## 问题

帖子的问题可以用`类型不同`和`指针在extern声明时不同于数组`来直接了当地解释。而类型不匹配如何导致程序崩溃的呢

```c
//file file1.c

int a[2] = {800, 801};
int b[2] = {100, 101};
```

```c
//file file2.c

extern int a[2];

// here b is declared as pointer,
// although the external unit defines it as an array
extern int *b; 

int main() {

  int x1, x2;

  x1 = a[1]; // ok
  x2 = b[1]; // crash at runtime, segment fault

  return 0;
}
```

当声明为`extern 数组`时，编译后符号表保存了数组的地址，那么a[1]解引用只是在数组地址上加偏移量`a+1`得到新地址，然后取新地址的内容，所以数组的地址提前可知  
当声明为`extern 指针`时，编译后符号表保存的是符号b，让编译器以指针的方式处理b，所以b是一个指针，其指针地址被赋值为数组的内容（101100）。在运行时，先取出b的地址为（101100），  
再加上偏移量得到新地址(101101), 最后访问这个非法地址导致崩溃

因为当声明为`extern int *b`时，在file1中的b应该是存放一个内存地址而实际存放的是数组内容，就导致程序把数组的内容当地址访问。

## 打印指针地址
如果在coredump之前，先打印b的地址，得到的是b数组的值为'0x6500000064', 正好指针长度是两个int之和。所以file2中把数组内容当成了指针地址，然后偏移一个int长度后，地址是'0x6500000068', 然后这个无效地址导致段错误。
```c
printf("%p\n", (void*)b);
```

## 从汇编角度看这个问题

b存放的是内容，而程序以为是地址。因为实际上编译器不会为数组额外分配空间来存储指针, 但是要为指针分配空间。
```c
int main() {
    int *a = 0;
    int b[4];
    *b = 0;
    // 为了显示栈大小
    foo();
}
```
从汇编代码可见，分配了32个字节的空间，指针8 + 4*4 = 
```
        sub     rsp, 32
        mov     QWORD PTR -8[rbp], 0
        mov     DWORD PTR -32[rbp], 0
```





## 弱符号
