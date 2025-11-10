---
title: cmake precompile header
date: 2022-02-23
tags: [cmake]
---

# cmake预处理头文件

预处理头文件是cmake 3.16添加的新功能，可以将头文件先

```
Precompiling header files can speed up compilation by creating a partially processed version of some header files, and then using that version during compilations rather than repeatedly parsing the original headers.
```
