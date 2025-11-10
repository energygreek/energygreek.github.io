---
date: 2020-07-20
tags: [c/cpp]
---


# C++ 小技巧

## 使用lambda 定义一个可变的const
因为const 不能改变的，只有在程序运行的时候初始化。 而通过lamda，可以通过获取命令行输入来改变值

```
const int x = []() -> int {
        int t;
        std::cin >> t;
        return t;
}();

```

