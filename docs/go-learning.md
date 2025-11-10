---
date: 2021-05-21
tags: [golang]
---

# go 学习笔记
## go为什么要指针和什么时候必须用指针
1. 传递waitgroup时
2. 获取命令行参数

## 全局变量和多变量赋值问题
因为很多函数需要处理error, 当存在全局变量时，可能会覆盖全局变量

```
var global_var int = 1
func foo(){
  // 此时覆盖了全局变量
  global_var, err := os.OpenFile("file")
}
```
应改为
```
var global_var int = 1
func foo(){
  var err error
  global_var, error = os.OpenFile("file")
}
```

## 数组和slice的区别
数组和slice使用上如同c++里的std::array和std::vector, 数组的长度固定，slice长度可以2倍扩容  
但有传参上有大区别: golang中数组是值传递，相当于将整个数组复制一遍，而slice是引用传递，所以slice用得多，map同理  

定义数组和slice:
```golang
// 数组
var a [3] int
arr := [5] int {1,2,3,4,5}
var array2 = [...]int {6,7,8}
q := [...] int {1,2,3} // 省略长度
q2 := [...] int {99, -1} // 前99项为0,第100个元素是-1

// slice
s1 := []int{1, 2, 3}
a := [10]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 0}  //a是数组
s2 := a[2:8]         
s3 := make([]int, 10, 20) 
```

## var 和 := 区别, 以及哪些类型需要make
两者很多时候可以相互替换，但是在不同类型上，有区别

对于基本类型string, int, boolean, 数组，var`声明`即初始化
```
var str string // 自动初始化为空字符串
var inter int // 自动初始化为0
var bo bool // 自动初始化为false

// 可以直接使用
fmt.Printf("%s %d %t", str, inter, bo) 
```

而对于slice, map, chann类型而言, 使用`var m map[int]string`只是声明，需要再用`make`来获得内存和初始化
```
var m map[int] string // 此时不能使用m
m = make(map[int] string){1:"a",2:"b"}
fmt.Println(m[1])
```
而上面的步骤可以简化成
```
m := make(map[int] string){1:"a", 2:"b"}
```
或直接
```
m := map[int] string{1:"a", 2:"b"}
```
