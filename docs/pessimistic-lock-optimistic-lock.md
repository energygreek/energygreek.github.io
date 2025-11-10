---
date: 2020-07-28
tags: [c,cpp,lock]
---

# 乐观锁悲观锁

先说这两种锁都不是编程里面用到的锁， 而是一种策略。 在数据库中用的多。  
而且在java里面有个原子bool AtomicBoolean 的实现也用到乐观锁的策略， `AtomicBoolean::compareandswap` 

这里以数据库管理服务（DBMS）来说明这两种锁的区别，在并发的情况下

## 悲观锁

悲观锁策略预期是多个请求会操作同一个数据，导致数据不可靠。 悲观锁的策略就是上锁，不让其它请求修改， 即排他锁。

这种排他锁与我们平常编程用的互斥锁道理相同， 是性能比较低的一种锁，但能保证数据准确。

## 乐观锁

乐观锁实际是不上锁， 预期多个请求不会修改同一分数据。而是在最后提交的时候再去确认数据是否一致，如果一致则提交，如果不一致则报错，需要认为干涉

所以乐观锁的数据也是可靠的， 且性能较好。 这里关键是如何确认数据一致  

## 乐观锁的确认并提交

这里举例sql的来说明 

```
select quanity from items where id =1;

//得到quanity=3，然后更新

update items set quantity=2 where id=2 and quantity=3

```
更新时，会确认quantity为预期的3，否则不执行或者报错。

这样有个问题， 虽然单条语句是原子的，但是两条语句不是， 会出现ABA的情况

例如当一个请求1执行到select后，得知`quantity=3`， 然后请求2先更新quantity=2，然后又更新为quantity=3 
然后请求1继续执行update，这会成功，虽然quantity=3是一致的，但是不能保证其它数据是一致的或者完全没有引起安全问题

因此另一种办法是判断只增version或者timestamp，每次请求时去更新。这两种办法都能确保数据一致。
但是因为数据库同时只允许一个请求修改version，那么就出现了竞争情况，会导致其它请求失败

例如多个请求同时记录了version=1, 则当一个请求完成的时候，version=2， 那么其它请求提交时发现verison变了，只能报告失败
虽然这样能保证数据正确，但是会导致其它请求失败

而电商平台中， 则采用另一种方式确认

```
select quanity from items where id =1;

//得到quanity=3，然后更新

update items set quantity=2 where id=2 and quantity > 0

```

这样就规避了上面两个方法的缺点

## compareandswap 

```
compareAndSet(expect_val, new_val)
```

上面的乐观锁就是compareandswap的解释：

先对比，如果一致则更新

再说一下java的实现,
```
    UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))  
      UnsafeWrapper("Unsafe_CompareAndSwapInt");  
      oop p = JNIHandles::resolve(obj);  
      jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);  
      return (jint)(Atomic::cmpxchg(x, addr, e)) == e;  
    UNSAFE_END  
```

在unsafe.cpp 找到实现如下

```c
#define LOCK_IF_MP(mp) __asm cmp mp, 0 \
__asm je L0 \
__asm _emit 0xF0 \
__asm L0:

int mp = os::is_MP();
__asm {
mov edx, dest
mov ecx, exchange_value
mov eax, compare_value
LOCK_IF_MP(mp)
cmpxchg dword ptr [edx], ecx
}
```
_emit 是用于生成指令的， 生成锁的指令，有cpu提供锁机制
从上下文看， 这个指令应该是上锁， 如果mp != 0 则上锁。 
