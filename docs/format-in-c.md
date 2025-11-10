---
comments: true
date: 2022-05-05
tags: [c]
---

# 在c语言中打印支持不同类型变量
in this doc, i'll introduce how to print(stringify) variable of different types, `typeof` is a GCC extension, and `_Generic` is part of C11. typeof is disabled by option '-std=*' or '-ansi`, so use `__typeof__` instead. 

## c11 _Generic

```c
#include <stdio.h>

#define PRINT_VAR(var)        \
    _Generic((var),           \
        int: print_int,       \
        float: print_float,   \
        double: print_double, \
        char: print_char      \
    )(#var, (var))

// Type-specific print functions
void print_int(const char* name, int value) {
    printf("%s = %d\n", name, value);
}

void print_float(const char* name, float value) {
    printf("%s = %.2f\n", name, value);
}

void print_double(const char* name, double value) {
    printf("%s = %.2f\n", name, value);
}

void print_char(const char* name, char value) {
    printf("%s = %c\n", name, value);
}

int main() {
    int x = 10;
    float y = 3.14f;
    double z = 3.14159;
    char c = 'A';

    PRINT_VAR(x);  // Prints: x = 10
    PRINT_VAR(y);  // Prints: y = 3.14
    PRINT_VAR(z);  // Prints: z = 3.14
    PRINT_VAR(c);  // Prints: c = A

    return 0;
}
```

## simpler way

linter raise warnings if place `printf` directly in `_Generic`.

```c
#define DECLARE_PRINT_FUNC(value)                                              \
  do {                                                                         \
    printf("%s = ", #value);                                                   \
    char *format = NULL;                                                       \
    _Generic((value),                                                          \
        int: format = "%d\n",                                                  \
        long unsigned int: format = "%lu\n",                                   \
        long: format = "%ld\n",                                                \
        float: format = "%.2f\n",                                              \
        double: format = "%.2f\n",                                             \
        char: format = "%c\n",                                                 \
        default: format = NULL);                                               \
    format != NULL ? printf(format, (value)):                                  \
    printf("Unknown type\n");                                                  \
  } while (0)

```

## using GCC build-in `typeof` instead of `_Generic`

```c
#include <stdio.h>

#define PRINT_VAR_TYPEOF(var)                                                  \
  do {                                                                         \
    printf("%s = ", #var);                                                     \
    char *format = NULL;                                                       \
    __typeof__(var) _v = (var);                                                \
    if (__builtin_types_compatible_p(__typeof__(_v), int))                     \
      format = "%d\n";                                                         \
    else if (__builtin_types_compatible_p(__typeof__(_v), float))              \
      format = "%.2f\n";                                                       \
    else if (__builtin_types_compatible_p(__typeof__(_v), double))             \
      format = "%.2f\n";                                                       \
    else if (__builtin_types_compatible_p(__typeof__(_v), char))               \
      format = "%c\n";                                                         \
    printf(format, (var));                                                     \
  } while (0)

```