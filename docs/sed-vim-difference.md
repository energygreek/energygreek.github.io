---
date: 2024-04-25
tags:
  - linux
  - tool
---

# sed和vim替换文本的小区别
## 替换招行的账单中发现vim和sed区别
在vim中
1. `:%s/\n/,/g`
2. `:% s/\t,\t,\t,\t,/\r/g`
用sed:
` cat bill.txt| sed -z 's/\n/,/g'  | sed 's/\t,\t,\t,\t,/\n/g'`

## 可见

2. vim中匹配换行符用`\n`, 而替换用`\r`。替换成`\n`在vim中显示成`^@`
1. sed 中的换行符统一用`\n`，这比vim好，但sed 中不能直接用`\n`匹配，还得加上 -z 参数。