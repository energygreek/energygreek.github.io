---
Gtitle: 模板编程(meta-programing)
date: 2021-03-04
tags: [c/cpp,template programing]
---


# 模板编程

模板编程是其它高级语言没有的技术, 也称为范型编程，元编程(meta programing), stl的基石。这种对类型的泛化相当于在编程之上编程

## 概述

模板可以作用于函数和类，即能泛化`类型`，也可以泛化`大小`
```c++
 template <typename T, int N> void foo() { T t[N]; }
 template <typename T, int N> struct Foo { T t[N]; };
  
 int main() {
    // array of two int
    foo<int, 2>();
    // class has a array of two int 
    Foo<int, 2> F;
  } 
```

## 模板的特化

跟泛化相反的方向，叫特化，编译器会优先使用特化的版本, 而特化有2个方向，`类型特化`和`范围特化`  

### 类型特化

对于特化类型个数的不同，分为偏特化和全特化，全特化即为所有类型都指定，特化类型越多匹配优先级越高 
注意：模板函数不能偏特化

```
template<typename T1, typename T2> struct Foo{};
template<typename T2> struct Foo<int, T2> {};
// 全特化
template<> struct Foo<int,int>{};
```

例如`Foo<int, int> foo'，编译器会使用第三个版本

### 范围特化

比如常见的指针和引用，这和int, float, class 都是无关的，属于另个维度，也可以说是范围, 在stl为兼容指针做大量的工作  

```
template <typename T, typename N> struct Foo<T *, N> {};
```
这样'Foo<int *, int> foo'会使用这个版本

还例如指定对大小的特例化
```
//模板
template<int n> foo(){}

//值特例化
template<> foo<10> foo(){}
```
那么如果调用`foo<10>();`时，优先匹配特例化版本

### 函数匹配优先级

在函数调用时，普通函数的匹配优先级高于模板函数
```
template <typename T> void f(T) { std::cout << "temp\n"; }
void f(int d) { std::cout << "temp1\n"; }
template <> void f(int d) { std::cout << "temp2\n"; }

f(1); // temp1
```

### 自定义类型的范围特化

上面讲的是指针类型和引用类型两种，但如果是自定义类型，那就无穷无尽了，所以模板编程也是'图灵完备'的

例如以下，创建了一个自定义的类型来包装基本类型(int,float)，这样可以有自定义类型的特化版本
```
  template <typename T> struct Decor { using type = T; };
  template <typename T> struct Strip { using type = T; };
  template <typename T> struct Strip<Decor<T>> { using type = T; };
  template <typename T> using StripDecor = typename Strip<T>::type;
  
  template <typename T> class Row {};
  
  int main() {
    using nodecor = Row<int>;
    using decor = Decor<Row<int>>;
    // 虽底层同为int, 但nodecor 类型不同于decor类型
    static_assert(std::is_same<col>, nocol>::value);
    // 通过Strip取出其底层类型
    static_assert(std::is_same<StripDecor<col>, nocol>::value);
    return 0;
  }
```

### 模板的声明定义分离
1. 由于template用来生成函数和类，所以编译器需要同时知道template的类型和其细节，所以模板函数不支持将定义放到源文件中
2. 而且编译器通常是以cpp为编译单元，当编译模板cpp时不知道调用cpp, 编译调用cpp时，不知道模板cpp。所以模板函数不支持将定义放到源文件中
3. 对于模板类可以将成员函数的定义放到源文件，但要为每个成员函数都添加'template'限定, ***而且要为实例添加特例化***。
4. 显式特例化支持只声明不定义，而在源文件中为每种所需的类型都特例化，即与3相同。其实显示特例化是不需要特例化而强制特例化。 

如下，模板类的定义放到cpp中，这样会报错，因为在编译call_foo.cpp时不知到模板定义, 因为没有生成过int版本的Foo。  
为此，必须在foo.cpp里添加`template class Foo<int>`, 如同4，实在吃力不讨好。
```c
// foo.h
    template<typename T>
    class Foo {
    public:
      Foo();
      void someMethod(T x);
    private:
      T x;
    };

// foo.cpp
    template<typename T>
    Foo<T>::Foo()
    {
      // ...
    }
    template<typename T>
    void Foo<T>::someMethod(T x)
    {
      // ...
    }

// call_foo.cpp
    void blah_blah_blah()
    {
      // ...
      Foo<int> f;
      f.someMethod(5);
      // ...
    }
```

当然把模板的定义放到头文件中会增加可执行文件的体积。

#### 实际运用

grpc 能以流和非流方式传输，而grpc参数protobuf消息类型是另一个范围。那么要封装grpc方法, 需要封装流+类型

举例protobuf的消息类型有string, fixed32
- string
- fixed32
- Stream<fixed32>
- Stream<string>

通过上面的偏特化可以即能区分流和非流又能区分类型

### 赘述一下模板的类型

上面的模板类型T都是实际编程时定义的类型，但作为图灵完备的模板编程，未决的template类型也可以作为template类型

如下是一个模板用另一个模板来特例化
```
template <typename T> struct Upper {};
template <template <typename> class T> struct Lower {};

template<typename T>
Lower<Upper<T>> l;
```

## 可变模版variadic templates

c中有可变参数`...`和gcc内置`__VA_ARGS__`宏定义, 实现不同个数的变量打印。这是由编译期实现的，会将format的格式符替换成参数
```
int printf ( const char * format, ... );
```

c++有模版，而且在c++11之后引入了动态参数模版，即模版函数或类可以使用动态参数

### 实际运用1

在实际项目中手动跑单元测试用例的时候，不希望再去看日志文件，而是想日志直接输出到终端, 有以下办法

#### c++11及以上标准

可以使用动态参数模版替换原本的日志打印函数
```c++
#undef log_debug

template<typename First, typename ...Rest>
void log_debug(First && first, Rest && ... rest){
  std::cout << fmt::format(first, rest ...) << std::endl;
}

// 这时日志就直接输出到终端了, 这里使用了fmt库
log_debug("aasdas{}", "bbbb");
```

也可简单写成
```c++
template<typename ...Args>
void log_debug(Args&& ...args){
	std::cout << fmt::format(args...) << std::endl;
}
```

如果不使用fmt格式化，还可以用
```c++
#undef log_debug
// 定义一个空函数
void log_debug(){}

template<typename First, typename Rest>
void log_debug(First&& first, Arg&& ...arg){
	std::cout << first;
	log_debug(arg...);
}
```
需要解释一下，函数`log_debug()`必须要先声明，因为模版实例化的的最终要调用这个无参的函数  
模拟一下堆栈, 因为参数在每次递归时减少一个，所以最终是0个参数
```
log_debug(1, 0.2, "aaa");
log_debug(0.2, "aaa");
log_debug("aaa");
log_debug();
```

#### c++11以前的标准

可以使用宏定义来替换了, 然后需要重载`,`方法
```c++
#define log_debug(...) std::cout , __VA_ARGS__ , std::endl

template <typename T>
std::ostream& operator,(std::ostream& out, const T& t) {
  out << t;
  return out;
}

//overloaded version to handle all those special std::endl and others...
std::ostream& operator,(std::ostream& out, std::ostream&(*f)(std::ostream&)) {
  out << f;
  return out;
}
```
#### 直接用c的方式

因为printf是支持varidic的
```c
#undef log_debug

#define log_debug(...) printf(__VA_ARGS__), printf("\n")

int main() {
  log_debug("example","output","filler","text");
  return 0;
}

```
#### c++17 引入了`fold expression` 

可以改写为
```c++
template<typename ...Args>
void log_debug(Args && ...args)
{
    (std::cout << ... << args);
}
```

动态参数模版除了以上的用法，还有更多用处, 例如`std::tupe`的实现

### 实际运用2

#### 使用模板生成并发代码

以下代码实现复制二维数组，
```
for( size_t ch=0 ; ch<channelNum ; ++ch ) {
    for( size_t i=0; i<length ; ++i ) {
        out[ch][i]=in[ch][i];
    }
}

```

但以下理论更快，没有两层for，前提是知道channel大小
```
for(size_t i=0;i<length;++i) {
    out[0][i]=in[0][i];
    out[1][i]=in[1][i];
    out[2][i]=in[2][i];
    out[3][i]=in[3][i];
}

```

但如果用模板，就不需要知道channel大小，自动生成上面的代码

```c++
template <int count> class Copy {
public:
  static inline void go(float **const out, float **const in, int i) {
    Copy<count - 1>::go(out, in, i);
    out[count - 1][i] = in[count - 1][i];
  }
};

template <> class Copy<0> {
public:
  static inline void go(float **const, float **const, int) {}
};

template <int channelNum>
void parall_copy(float **out, float **in, size_t length) {
  for (size_t i = 0; i < length; ++i) {
    Copy<channelNum>::go(out, in, i);
  }
}
```

需要提醒的，如同打印日志的，0的特例化不能省，否则编译出错

### 实际运用3 工厂模式

在使用spdlog时发现有使用到template未决名, 用来实现两个维度的工厂模式。

第一层提供两种sink的工厂，而其factory是未决名，所以要加上`Factory::template`消歧义，不然`<`会当成小于号  

```c
//
// factory functions
//
template<typename Factory = default_factory>
inline std::shared_ptr<logger> basic_logger_mt(const std::string &logger_name, const filename_t &filename, bool truncate = false)
{
    return Factory::template create<sinks::basic_file_sink_mt>(logger_name, filename, truncate);
}

template<typename Factory = default_factory>
inline std::shared_ptr<logger> basic_logger_st(const std::string &logger_name, const filename_t &filename, bool truncate = false)
{
    return Factory::template create<sinks::basic_file_sink_st>(logger_name, filename, truncate);
}
```

此时factory可以是`synchronous_factory`, 也可以是异步版本，但这需要用户自己实现。
```c
// Default logger factory-  creates synchronous loggers
struct synchronous_factory
{
    template<typename Sink, typename... SinkArgs>
    static std::shared_ptr<spdlog::logger> create(std::string logger_name, SinkArgs &&... args)
    {
        auto sink = std::make_shared<Sink>(std::forward<SinkArgs>(args)...);
        auto new_logger = std::make_shared<logger>(std::move(logger_name), std::move(sink));
        details::registry::instance().initialize_logger(new_logger);
        return new_logger;
    }
};

using default_factory = synchronous_factory;

```

简化为下面的demo，固然可以直接使用call_dd的方式，但维度只有一个。而call_dd2则有2个维度了  
但必须使用`T::template`消歧义, 因为此时的foo未决名, 不知道是那个类里面的foo。
```
  struct AA {
    template <typename cc> static void foo() { std::cout << "dd::foo\n"; };
  };
  
  struct BB {
    template <typename cc> static void foo() { std::cout << "dd::foo\n"; };
  };
  
  template <typename T> void call_dd() { AA::foo<T>(); }
  template <typename T, typename K> void call_dd2() { T::template foo<K>(); }
  
  int main() {
    call_dd<void>();
    call_dd2<AA>();
    call_dd2<AA, int>();
    call_dd2<BB, long>();
  }
```

##模版与宏定义、虚函数的区别

1. 宏定义在预处理期执行，模板在编译期执行，而虚函数也称动态绑定在运行时执行
2. 宏和模板都将运行时的工作提前了，用编译时间换取运行效率
3. 宏定义没有类型检查，这点模板比较好
4. 模板虽然会延长编译时间，但当编译期实例化类型后，查找模板函数和查找普通函数的速度几乎相同

## 待决名dependent name
1. 待决名的意思是在定义的地方，类型还不能决断，需要延后到实例化确定时。而非待决名指类型在定义的地方已经确定。  
2. 延后将导致此时无法在定义点进行错误检查，以及消除`typename`和`template`歧义，这导致需要在调用点加上template

待决名如:
```
template<typename T>
struct X : B<T> // "B<T>" 取决于 T
{
    typename T::A* pa; // "T::A" 取决于 T
                       // （此 "typename" 的使用的目的见下文）
    void f(B<T>* pb)
    {
        static int i = B<T>::i; // "B<T>::i" 取决于 T
        pb->j++; // "pb->j" 取决于 T
    }
};
```

让人吃惊的例子, 这就是非待决名的情况下，立即绑定
```c++
#include <iostream>
 
void g(double) { std::cout << "g(double)\n"; }
 
template<class T>
struct S
{
    void f() const
    {
        g(1); // "g" 是非待决名，现在绑定
    }
};
 
void g(int) { std::cout << "g(int)\n"; }
 
int main()
{
    g(1);  // 调用 g(int)
 
    S<int> s;
    s.f(); // 调用 g(double)
}
```

### typename消歧义

在模板（包括别名模版）的声明或定义中，不是当前实例化的成员且取决于某个模板形参的名字不会被认为是类型，  
除非使用关键词 typename 或它已经被设立为类型名（例如用 typedef 声明或通过用作基类名）。 

```c
#include <iostream>
#include <vector>
 
int p = 1;
 
template<typename T>
void foo(const std::vector<T> &v)
{
    // std::vector<T>::const_iterator 是待决名，
    typename std::vector<T>::const_iterator it = v.begin();
 
    // 下列内容因为没有 'typename' 而会被解析成
    // 类型待决的成员变量 'const_iterator' 和某变量 'p' 的乘法。
    // 因为在此处有一个可见的全局 'p'，所以此模板定义能编译。
    std::vector<T>::const_iterator* p; 
 
    typedef typename std::vector<T>::const_iterator iter_t;
    iter_t * p2; // iter_t 是待决名，但已知它是类型名
}
 
template<typename T>
struct S
{
    typedef int value_t; // 当前实例化的成员
    void f()
    {
        S<T>::value_t n{}; // S<T> 待决，但不需要 'typename'
        std::cout << n << '\n';
    }
};
 
int main()
{
    std::vector<int> v;
    foo(v); // 模板实例化失败：类型 std::vector<int> 中没有
            // 名字是 'const_iterator' 的成员变量
    S<int>().f();
}
```

### template消歧义

与此相似，模板定义中不是当前实例化的成员的待决名同样不被认为是模板名，除非使用消歧义关键词 template，或它已被设立为模板名： 

```
template<typename T>
struct S
{
    template<typename U> void foo() {}
};
 
template<typename T>
void bar()
{
    S<T> s;
    s.foo<T>();          // 错误：< 被解析为小于运算符
    s.template foo<T>(); // OK
}
```
template 消歧义可以使用
```
T::template
s.template
this->template
```

## std::forward 转发在模版的使用

为什么完美转发的对象必须是右值引用？

### 说明一下右值引用

```
 引用类型 	可以引用的值类型 	使用场景
非常量左值 	常量左值 	非常量右值 	常量右值
非常量左值引用 	Y 	N 	N 	N 	无
常量左值引用 	Y 	Y 	Y 	Y 	常用于类中构建拷贝构造函数
非常量右值引用 	N 	N 	Y 	N 	移动语义、完美转发
常量右值引用 	N 	N 	Y 	Y 	无实际用途
```

##参考
https://en.cppreference.com/w/cpp/language/parameter_pack
https://en.cppreference.com/w/cpp/language/fold
https://en.cppreference.com/w/cpp/language/overload_resolution#Best_viable_function

