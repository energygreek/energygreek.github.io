---
date: 2022-03-10
tags: [c/cpp, atomic]
---

# CPP原子操作

## 无锁队列

## atomic

```cpp
      void
      store(__pointer_type __p,
	    memory_order __m = memory_order_seq_cst) noexcept
      { return _M_b.store(__p, __m); }

      void
      store(__pointer_type __p,
	    memory_order __m = memory_order_seq_cst) volatile noexcept
      { return _M_b.store(__p, __m); }

      __pointer_type
      load(memory_order __m = memory_order_seq_cst) const noexcept
      { return _M_b.load(__m); }

      __pointer_type
      load(memory_order __m = memory_order_seq_cst) const volatile noexcept
      { return _M_b.load(__m); }

      __pointer_type
      exchange(__pointer_type __p,
	       memory_order __m = memory_order_seq_cst) noexcept
      { return _M_b.exchange(__p, __m); }
```
## cpp实现

```
atomic::compare_exchange_weak
```

## memory_order

* memory_order_relaxed
* memory_order_seq_cst
