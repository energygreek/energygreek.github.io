---
date: 2020-07-23
tags: [safety]
---

# 主要收集一些常见缺陷和加固方法

## javascript

1. xss漏洞：
在输入框注入代码如 `<script> alert(1)</script>`   
解决办法是校验输入
