---
date: 2022-03-11
tags: [c]
---

# 编程语言中小数精度问题

计算机使用2进制存储数据，而2进制可以表示所有的10进制整数，这样在整数范围内不存在问题，但导致小数的精度不够。

例如表示整数时，二进制不存在问题: 
|十进制|二进制|
|--|--|
|0| 0|
|1|2^0|
|2|2^1|
|3|2^2 + 2^1|

而表示小数时，二进制只能有0.5, 0.25, 0.125...。
|二进制|十进制|
|--|--|
|2^-1| 0.5|
|2^-2|0.25|
|2^-3|0.125|

所以二进制表示10进制的小数颗粒比较大，若想表示0.6: 就要使用`二进制小数` `*` `2^n`来表示，有些数始终无法相等

## 浮点型和双精度

计算机显然没有无穷大的位置来存储”高阶的X“，只能折衷一下。故在c/c++中，有两种类型来表示小数，对应不同精度:    
内存分布，两种类型的内存分布都为: 符号位(sign)+偏移位(exponent)+尾数位(mantissa)，
其值则为 (-1)^sign * mantissa * 2^exponent

### float: 浮点类型
占用4字节（32位），精度为10^-6, 即第7位小数开始可能四舍五入或者直接忽略

内存分布： 第一位是符号位，后8位为偏移位，最后23位为尾数位，则有23+1位来表示小数，10^7 < 2^23+1 < 10^8，所以可以精确到第7位，
但最后以为可能会4四舍五入，精确到第6位

### double: 双精度类型
占用8字节（64位），精度为10^-15, 即第16位小数开始可能四舍五入或者直接忽略

内存分布： 第一位是符号位，后11位为偏移位，最后52位为尾数位，则有52+1位来表示小数，10^16 < 2^52+1 < 10^17，所以可以精确到第16位，
但最后以为可能会4四舍五入，精确到第15位

## 计算问题

由于精度问题，精度之外的小数就会被忽略掉，导致出现下面的相等 

```
double i = 0.1;
double j = 0.2;
i+j == 0.3 + 4e-17;// 后面的数被忽略了
```
注意如果要使用float类型，需要显式使用'0.1f', 因为默认以double处理  

## 如何比较两个浮点数

浮点数比较不可能100%正确，但下面都是错的
```
float a = 0.15 + 0.15
float b = 0.1 + 0.2
if(a == b) // can be false!
if(a >= b) // can also be false!
if( Math.abs(a-b) < 0.00001) // wrong - don't do this
if( Math.abs((a-b)/b) < 0.00001 ) // still not right!
```

下面的才是大部分正确
```c
float absA = fabs(a);
float absB = fabs(b);
float diff = fabs(a - b);
if (a == b) { // shortcut, handles infinities
	return true;
} else if (a == 0 || b == 0 || (absA + absB < 1e-16)) {
			// a or b is zero or both are extremely close to it
			// relative error is less meaningful here
	return diff < (epsilon * Float.MIN_NORMAL);
} else { // use relative error
	return diff / Math.min((absA + absB), Float.MAX_VALUE) < epsilon;
}
```

但如果知道精度，直接将其乘以10^n变成整数来比较是最佳的方法