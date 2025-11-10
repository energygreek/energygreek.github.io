---
date: 2021-03-15
tags: [cpp]
---

# 智能指针

智能指针是c++11加入的特性，包括shared_ptr和unique_ptr，weak_ptr，以及make_shared函数，但make_unique是c++14才出来  
不过c++11可以通过模版来实现make_unique

## 普通指针和智能指针的区别

shared_ptr和unique_ptr都是异常安全的(exception-safe)，而普通指针不是，举例如下
```
void nonsafe_call(T1* t1, T2* t2);
void safe_call(unique_ptr<T1> t1, unique_ptr<T2> t2);

nonsafe_call(new T1, new T2);
safe_call(make_unique<T1>, make_unique<T2>);
```
### 引用计数
![[shared_ptr.png]]
### 当形参为普通指针时

虽然new是安全的，会先申请内存再构造对象t1。 如果t1的构造函数抛异常，申请的内存会自动释放，不会内存泄漏  
但当两个new作为函数参数时，情况不同。 因为参数必须在调用函数前决断，所以步骤如下
```
1. 为t1申请内存
2. 构造t1
3. 为t2申请内存
4. 构造t2
5. 调用函数
```

- 当执行到2失败时，不会泄漏，new会释放t1的内存
- 当执行到4失败是，会泄漏t1，因为t1已经构造完成，不会释放内存

### 当形参为智能指针时

步骤为
```
1. t1 = make_unique<T1>()
2. t2 = make_unique<T2>()
3. 调用函数

```
如果2失败，t1的对象由unique_ptr管理，当t1释放时，会释放内存，所以无论t1构造失败还是t2构造失败，都能正确释放内存，不会导致泄漏  
就因为unique_ptr和shared_ptr是内存安全的

### 但以普通指针构造智能指针的方式不是异常安全的

即`foo(std::unique_ptr<T1>(new T1()), std::unique_ptr<T2>(new T2()));` 不是异常安全的

此时步骤可能如下
```
1. 为t1申请内存
2. 构造t1
3. 为t2申请内存
4. 构造t2
5. 构造unique_ptr<T1>
6. 构造unique_ptr<T2>
7. 调用函数
```
当步骤4发生异常时，t1并未被unique_ptr管理，所以不会去释放t1的内存，故发生内存泄漏

## 智能指针的构造

shared_ptr对应的函数是make_shared<T>  
但c++11中没有make_unique, 可以使用模版实现如下

```c
template<typename T,typename ...Args>
std::unique_ptr<T> make_unique(Args&& ...args){
    return std::unique_ptr<T>(new T(std::forward<Args>(args)... ));
}
```
那为什么这样就能异常安全呢？因为调用make_uniqe时，可以确保t1的内存被unique_ptr管理了 

## make_shared创建智能指针和用new创建智能指针的区别

除了上面说所的make_shared/make_unique是异常安全，而unique_ptr<T>(new T()) 不是异常安全外，其构造的智能指针也不同  

智能指针之所以会释放内存，是因为智能指针本身也是一个对象，在其生命周期结束后会调用dtor, 并在那里释放内存  
所以智能指针的对象内存即包含了T，也包含了智能指针本身，而make_shared<T>() 和 shared_ptr<T>(new T)的区别在于`前者是两个部分的地址连续，而后者并不一定连续`

## unique_ptr的区别

unique_ptr和shared_ptr不同，后者为了多变量同时拥有资源的访问，而unique_ptr表示任何时刻，只有一个变量能访问资源和释放资源，比如打开的文件设备, 
而且通常为管理的资源设置析构函数，比如指定析构时关闭文件和设备

如下代码，当fp析构时，自动关闭文件
```
// helper function for the custom deleter demo below
void close_file(std::FILE* fp)
{
    std::fclose(fp);
}
using unique_file_t = std::unique_ptr<std::FILE, decltype(&close_file)>;
unique_file_t fp(std::fopen("demo.txt", "r"), &close_file);
```

unique_ptr要配合move使用，当move之后，原对象不拥有指针资源

## 总结

为了避免多参数决断时，导致已经决断的参数内存泄漏，应尽可能使用智能指针来管理内存
