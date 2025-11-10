---
date: 2021-12-09
tags: [算法,DP]
---

# DP 四板斧

以跳台阶的题目为例， 题目是以每次跳1层或2层的方式跳到n台阶， 问一共有多少种跳法  
开始我觉得这是求路径个数，而通常dp算最大或最小值， 不适用。   
但后来换个角度想，求最大最小必须得试过所有方法才得出来的， 而将所有的试过的方法计数，就是这题的解

## 写出递归

dp第一步要先写出递归来， 写递归得写收敛条件  
这里的收敛条件是当传参小于0,则说明解法失败（多跳了），返回0。 当恰好为0则解法有效返回1  

递归体就是跳1层和跳2层

```c
static int jump(int number) {
  if (number < 0) {
    return 0;
  } else if (number == 0) {
    return 1;
  }
  return jump(number - 1) + jump(number - 2);
}
  
int jumpFloor(int number) { 
  return jump(number - 1) + jump(number - 2); 
}
```

## 用缓存来优化递归

递归显然会重复计算， 可以用缓存计算结果来避免， 当有缓存时使用缓存，否则才去计算， 最后将结果缓存  

并且使用缓存时应该指定尾递归的值

```c
int jump(int number, int *cache) {
  if (number < 3) {
    return cache[number];
  }
  // 缓存没命中就计算
  int ret1 =
      cache[number - 1] != -1 ? cache[number - 1] : jump(number - 1, cache);
  int ret2 =
      cache[number - 2] != -1 ? cache[number - 2] : jump(number - 2, cache);
  cache[number] = ret1 + ret2;
  return cache[number];
}

int jumpFloor(int number) {
  int cache[number + 1];
  for (int i = 0; i < number + 1; i++) {
    cache[i] = -1;
  }

  cache[0] = 0;
  cache[1] = 1;
  cache[2] = 2;
  return jump(number, cache);
}
```

## 递归改for循环

递归的调用流程是自上而下，先计算(n)->计算(n -1)->再计算(n-2) ... 0  
现在要反过来， 先计算(0)->计算(1)->计算(2) ... n 
即自下而上

```c
int jumpFloor(int number) {
  if (number < 3) {
    return number;
  }

  // 之前要初始化为了判断， 而现在显然下标大于2的值是要计算出来的，不需要读取  
  // 所以不需要初始化
  int cache[number + 1];
  cache[1] = 1;
  cache[2] = 2;

  for (int i = 3; i <= number; i++) {
    cache[i] = cache[i - 1] + cache[i - 2];
  }

  return cache[number];
}
```

## 缩小缓存大小

仔细看， 其实每次计算只涉及最大的三个数， 所以数组并不需要那么大，只用3个单元就足够，故直接写成3个变量

```c
  int jumpFloorD(int number) {
    if (number < 3) {
      return number;
    }
  
    int total = 0;
    int minors2 = 1;
    int minors1 = 2;
  
    for (int i = 3; i <= number; i++) {
      total = minors1 + minors2;
      minors2 = minors1;
      minors1 = total;
    }
  
    return total;
  }
```

这样DP解法就出来了， 这个套路是我从力扣上学到的， 非常有效简单

## 总结

从上可见，你需要训练如何发现递归关系，剩下的工作就是缓存递归调用的结果，最后是将从上到下的解决思路变为从下到上。这样我们就可以用O(N)的时间复杂度去遍历  

## 参考

https://leetcode.com/discuss/study-guide/1490172/dynamic-programming-is-simple
