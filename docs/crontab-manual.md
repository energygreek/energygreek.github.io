---
date: 2021-08-24
tags: [linux]
---

# Linux定时器
https://www.computerhope.com/unix/ucrontab.htm#command-entries


## 使用systemd-run 替换

```
# systemd-run --on-active=30 /bin/touch /tmp/foo
# systemd-run --on-active="12h 30m" --unit some-unit.service
```
