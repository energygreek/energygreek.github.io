---
draft: true
comments: true
date: 2025-08-04
tags: [ip,network]
---

# learn ip header
ipv4的头最小长度是20, 因为ip4的头尾部可以包含一个或多个option和padding, 其长度可变。 而ipv6的头虽然固定长度，但是ipv6支持拓展 extension，其长度也可以变。


## ipv6 extension
ipv6的头里 'Next Header' 字段表示是否有下一个头。

## ipv4 options
* 记录路由路径
* 指定路由，曾经看到博客有人用路由打印出简历

## 参考
https://pall.as/ipv6-headers-and-extension-headers/
https://www.geeksforgeeks.org/computer-networks/options-field-in-ipv4-header/
