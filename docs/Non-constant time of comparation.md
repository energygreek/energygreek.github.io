---
comments: true
date: 2025-07-22
tags: [safety]
---

# Non-constant time comparison bug
Tailscale 给我发的邮件说到了这个BUG, 发现挺有趣的。

> Hello,
We recently fixed an insecure non-constant time comparison bug in the Tailscale DERP server that could have caused DERP servers to leak their private mesh keys via a timing side channel. We have deployed this fix to all Tailscale-operated DERP servers and rotated their mesh keys. 

## BUG原因
非固定时长的比较，就是比较次数约多，耗时越长，例如常见for循环，还有'memncmp'。利用这个时间差异，也许能够猜到密码。
```c
for (i = 0; i < len; i++) {
    if (a[i] != b[i]) return false;
}
```

## 解决办法
使用固定时间（constant time）的比较函数，例如改成

```c
flag = true
for (i = 0; i < len; i++) {
    if (a[i] != b[i]) flag = false;
}
return flag;
```
