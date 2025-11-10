---
date: 2021-09-22
tags: [cpp, string]
---

# c++的basic_string优化

Small-string optimization is basically 

```c++
union { 
  struct { char*, size_t, size_t; }
  struct { char[23]; bool; } 
} 
```
- but with bit-tricks instead.
When allocated, the local buffer's space is reused for size and capacity. It's not
wasted.
But gdb just shows both sides of the union, which can be confusing.
