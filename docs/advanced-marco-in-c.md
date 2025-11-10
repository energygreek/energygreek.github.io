---
title: advanced marco in c
date: 2020-11-10
tags: [c/cpp]
---

# C的宏定义

宏定义是在c/c++里特有的方式， 像变量一样， 又像模板编程一样， 但最常见的用法还是做头文件的唯一性保证  

在每一个头文件都套用这种格式，就可以避免多次引入头文件而导致的重复定义报错

```c
#ifdef FILE_NAME
#def FILE_NAME

// 代码

#endif FILE_NAME
```

## 原理

宏定义与变量、模板的最大区别在与处理的时期， 宏定义在预编译时处理， 而变量和模板函数则是在编译期处理。
查看预编译后的代码可以使用命令`gcc -E  ` 或者 `cpp `， 实际上是前者是调用了后者

```
NAME
       cpp - The C Preprocessor
```

## 用法

除了`#ifdef` 的用法，宏定义可以分两种类型，变量型和函数型

###  变量型

这个最简单，就像使用变量一样，先define 然后再使用

```c
# marco.c
#define BUFFER_SIZE 1024

int main(){
	foo = (char *) malloc (BUFFER_SIZE);
}
```
执行 `gcc -E marco.c` 得到
```c
foo = (char *) malloc (1024);
```

多行使用 '\' 来连接
```c
#include <stdio.h>
#define GREETING_STR \
  "hello \
world"
```

* 注意, 宏定义的定义不分前后， 也不像变量那样先定义再使用， 宏定义可以先使用后定义  

以下两种方式的效果相同

```c
#define GREETING_NAME "wayou"

#define GREETING "hello," GREETING_NAME

int main() {
printf(GREETING);
return 0;
}
```

```diff
+#define GREETING "hello," GREETING_NAME

#define GREETING_NAME "wayou"

-#define GREETING "hello," GREETING_NAME

int main() {
printf(GREETING);
return 0;
}
```

### 函数型  

函数类型的宏，可以像正常函数一样指定入参，入参需为逗号分隔合法的 C 字面量。
宏的参数必须要用括号包起来，否则当参数为表达式时，会出错

```c
#define min(X, Y)  ((X) < (Y) ? (X) : (Y))
  x = min(a, b);          →  x = ((a) < (b) ? (a) : (b));
  y = min(1, 2);          →  y = ((1) < (2) ? (1) : (2));
  z = min(a + 28, *p);    →  z = ((a + 28) < (*p) ? (a + 28) : (*p));
 ```

### 宏定义字符串化

当宏定义的参数被引号包起来时， 不会进行替换，如下
```c
#define foo(x) x, "x"
foo(bar)        → bar, "x"
```

加入需要将参数替换到字符串里， 可以使用'#'  

```c
#define WARN_IF(EXP) \
do { if (EXP) \
        fprintf (stderr, "Warning: " #EXP "\n"); } \
while (0)
WARN_IF (x == 0);
     → do { if (x == 0)
           fprintf (stderr, "Warning: " "x == 0" "\n"); } while (0);
```

而当 这里的x 也是宏定义时， 只有if里的x会替换， 字符串里的x则不会替换  

```c
#define X ( 1 - 1 )
WARN_IF ( X == 0);
```
会被替换为  

```c
do { if (( 1 - 1 ) == 0) fprintf (
stderr
, "Warning: " "X == 0" "\n"); } while (0);

```

### 拼接

通过 ## 可将两个宏展开成一个，即将两者进行了拼接，宏拼接一般用在需要拼接的宏是来自宏参数的情况，  
其他情况，大可直接将两个宏写在一起即可

当有以下情况时非常有用
```
struct command
{
  char *name;
  void (*function) (void);
};

struct command commands[] =
{
{ "quit", quit_command },
{ "help", help_command },
…
};
```

可以使用如下：
```c
#define COMMAND(NAME)  { #NAME, NAME ## _command }

struct command commands[] =
{
COMMAND (quit),
COMMAND (help),
…
};
```

### 不定参数和混合参数

宏定义也可以使用不定参数
```c
#define eprintf(args…) fprintf (stderr, args)
// or
#define eprintf(…) fprintf (stderr, __VA_ARGS__)
```

也可以使用混合参数
```c
#define eprintf(format, args...) fprintf (stderr, format, args)
```
这个可以常在格式化打印时用到， 例如 `spdlog` 库  

```c
#define SPDLOG_LOGGER_CALL(logger, level, ...)                                                                                             \
    if (logger->should_log(level))                                                                                                         \
    logger->log(spdlog::source_loc{SPDLOG_FILE_BASENAME(__FILE__), __LINE__, SPDLOG_FUNCTION}, level, __VA_ARGS__)

#if SPDLOG_ACTIVE_LEVEL <= SPDLOG_LEVEL_TRACE
#define SPDLOG_LOGGER_TRACE(logger, ...) SPDLOG_LOGGER_CALL(logger, spdlog::level::trace, __VA_ARGS__)
#define SPDLOG_TRACE(...) SPDLOG_LOGGER_TRACE(spdlog::default_logger_raw(), __VA_ARGS__)
#else
#define SPDLOG_LOGGER_TRACE(logger, ...) (void)0
#define SPDLOG_TRACE(...) (void)0
#endif

#if SPDLOG_ACTIVE_LEVEL <= SPDLOG_LEVEL_DEBUG
#define SPDLOG_LOGGER_DEBUG(logger, ...) SPDLOG_LOGGER_CALL(logger, spdlog::level::debug, __VA_ARGS__)
#define SPDLOG_DEBUG(...) SPDLOG_LOGGER_DEBUG(spdlog::default_logger_raw(), __VA_ARGS__)
#else
#define SPDLOG_LOGGER_DEBUG(logger, ...) (void)0
#define SPDLOG_DEBUG(...) (void)0
#endif

```

## 重复和覆盖

这些是相似的：
```c
#define FOUR (2 + 2)
#define FOUR         (2    +    2)
#define FOUR (2 /* two */ + 2)
```

以下都是不同的宏
```c
#define FOUR (2 + 2)
#define FOUR ( 2+2 ) // 空白位置不一样 
#define FOUR (2 * 2) // 宏的内容不一样
#define FOUR(score,and,seven,years,ago) (2 + 2) // 入参不一样
```

对于使用了 `#undef` 注销过的宏，再次定义同名的宏时，要求新定义的宏不与老的相似。

而如果说一个已经存在的宏，并没有注销，重复定义时，如果相似，则新的定义会忽略，如果不相似，编译器会报警告同时使用新定义的宏。这允许在多个文件中定义同一个宏。

## 最后

 可以查看更多[内置宏定义](https://gcc.gnu.org/onlinedocs/cpp/Predefined-Macros.html#Predefined-Macros)

