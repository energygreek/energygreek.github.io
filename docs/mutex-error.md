---
tags:
  - mutex
  - c
  - cpp
date: 2022-01-25
---


# mutex error
记一次单元测试用例时失败，且出现奇怪的报错
```
tpp.c:63: __pthread_tpp_change_priority: Assertion `new_prio == -1 ||(new_prio >= __sched_fifo_min_prio && new_prio <=__sched_fifo_max_prio)' failed
```

## 原因

函数定义如下，原因是下面的第一个assert条件不成立，new_prio != -1。这个函数的注释说明了需要使用者确保调用前已初始化。

```
/* We only want to initialize __sched_fifo_min_prio and __sched_fifo_max_prio
   once.  The standard solution would be similar to pthread_once, but then
   readers would need to use an acquire fence.  In this specific case,
   initialization is comprised of just idempotent writes to two variables
   that have an initial value of -1.  Therefore, we can treat each variable as
   a separate, at-least-once initialized value.  This enables using just
   relaxed MO loads and stores, but requires that consumers check for
   initialization of each value that is to be used; see
   __pthread_tpp_change_priority for an example.
 */
```

```c
int
__pthread_tpp_change_priority (int previous_prio, int new_prio)
{
  struct pthread *self = THREAD_SELF;
  struct priority_protection_data *tpp = THREAD_GETMEM (self, tpp);
  int fifo_min_prio = atomic_load_relaxed (&__sched_fifo_min_prio);
  int fifo_max_prio = atomic_load_relaxed (&__sched_fifo_max_prio);
  if (tpp == NULL)
    {
      /* See __init_sched_fifo_prio.  We need both the min and max prio,
         so need to check both, and run initialization if either one is
         not initialized.  The memory model's write-read coherence rule
         makes this work.  */
      if (fifo_min_prio == -1 || fifo_max_prio == -1)
        {
          __init_sched_fifo_prio ();
          fifo_min_prio = atomic_load_relaxed (&__sched_fifo_min_prio);
          fifo_max_prio = atomic_load_relaxed (&__sched_fifo_max_prio);
        }
      size_t size = sizeof *tpp;
      size += (fifo_max_prio - fifo_min_prio + 1)
              * sizeof (tpp->priomap[0]);
      tpp = calloc (size, 1);
      if (tpp == NULL)
        return ENOMEM;
      tpp->priomax = fifo_min_prio - 1;
      THREAD_SETMEM (self, tpp, tpp);
    }
  assert (new_prio == -1
          || (new_prio >= fifo_min_prio
              && new_prio <= fifo_max_prio));
  assert (previous_prio == -1
          || (previous_prio >= fifo_min_prio
              && previous_prio <= fifo_max_prio));

```

网上帖子的原因是给初始化mutex并设置RECURSIVE类型时，忘了对mutexattr进行初始化，导致使用了脏内存
即`pthread_mutexattr_init(&mutexattr)`

```
pthread_mutexattr_t mutexattr;

// Set the mutex as a recursive mutex
pthread_mutexattr_settype(&mutexattr, PTHREAD_MUTEX_RECURSIVE_NP);

// create the mutex with the attributes set
pthread_mutex_init(&CritSec, &mutexattr);

// After initializing the mutex, the thread attribute can be destroyed
pthread_mutexattr_destroy(&mutexattr);
```

总之，要在线程启动前初始化好mutex

## 其他问题

这个问题其实是c++代码抛出来的，多线程执行了下面的代码，导致了上面的问题。

AppCache中有个小粒度的锁，但这个锁还未初始化就被其他线程去lock了，导致出现问题。所以要mutex最好要在线程启动前初始化

```
AppCache{
std::mutex
}；
```

```c++
  auto app = m_app_cache.find(app_path);
  if (app == m_app_cache.end()) {
    // create a new monitor element
    m_app_cache.insert_or_assign(app_path, std::make_shared<AppCache>());
  }
  std::lock_guard<std::mutex> lk(app->second->cache_mutex);
```


## 引用

https://sourceware.org/legacy-ml/libc-help/2008-05/msg00072.html?utm_source=pocket_mylist
