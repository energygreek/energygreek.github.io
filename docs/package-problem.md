---
date: 2022-05-12
tags: [算法,dp]
---

# 背包问题
有一个承重为V的背包，有N个物品，重量分别为Cost[i], 价值分别为Value[i], 问选择放哪些物品时包的价值最大

## 状态方程

解决动态编程的关键是找到状态方程，如下。 解决背包问题就是比较`放`与`不放`哪个价值高
```
F [i, v] ← max{F [i − 1, v], F [i − 1, v − Ci] + Wi}
```
## 01背包

要求物品最大为1个, 这里有几个版本

```c
#include <stdio.h>
#include <sys/param.h>

const int V = 4, N = 3;
const int cost[N] = {1, 3, 4};
const int value[N] = {15, 20, 30};

// const int V = 10, N = 5;
// const int cost[N] = {2, 3, 5, 2, 4};
// const int value[N] = {2, 2, 1, 1, 2};

void O1Package_N() {
  int dp[N][V + 1];

  // 0容量的价值为0, 合法
  for (int i = 0; i <= V; i++) {
    dp[0][i] = 0;
  }

  // 初始化当包承重大于物品0重量的情况
  for (int j = cost[0]; j <= V; j++) {
    // 当只装物品0时，包的价值就是物品0的价值
    dp[0][j] = value[0];
  }

  // 先遍历物品，后遍历重量
  for (int i = 1; i < N; i++) {
    for (int v = 0; v <= V; v++) {
      // F [i, v] ← max{F [i − 1, v], F [i − 1, v − Ci] + Wi}
      // F[i][v] = MAX(F[i - 1][v], F[i - 1][v - cost[i]] + value[i]);
      int bufang = dp[i - 1][v];
      dp[i][v] = bufang;
      if (v >= cost[i]) {
        int fang = dp[i - 1][v - cost[i]] + value[i];
        dp[i][v] = MAX(bufang, fang);
      }
    }
  }

  for (int i = 0; i < N; i++) {
    for (int j = 0; j <= V; j++) {
      printf("%d\t", dp[i][j]);
    }
    printf("\n");
  }
  printf("%d\n", dp[N - 1][V]);
}

void O1Package_N_plus_1() {

  int dp[N + 1][V + 1];

  // 使用0个物品的价值为0
  for (int i = 0; i <= V; i++) {
    dp[0][i] = 0;
  }

  // 先遍历物品，后遍历重量
  for (int i = 1; i <= N; i++) {
    for (int v = 0; v <= V; v++) {
      dp[i][v] = dp[i - 1][v]; //当重量不够装i时，价值为装i-1的价值,
                               //此步骤以备i+1时使用（下一个i循环）
      if (v >= cost[i - 1]) {
        dp[i][v] = MAX(dp[i - 1][v], dp[i - 1][v - cost[i - 1]] + value[i - 1]);
      }
    }
  }

  for (int i = 0; i <= N; i++) {
    for (int j = 0; j <= V; j++) {
      printf("%d\t", dp[i][j]);
    }
    printf("\n");
  }
  printf("%d\n", dp[N][V]);
}

void O1Package_N_plus_1_origin() {

  int dp[N + 1][V + 1];

  // 使用0个物品的价值为0
  for (int i = 0; i <= V; i++) {
    dp[0][i] = 0;
  }

  // 先遍历物品，后遍历重量
  for (int i = 0; i < N; i++) {
    for (int v = 0; v <= V; v++) {
      // 1. 当物品大于承重，物品不会放到背包，价值等于不放此物品时的情况
      // 即使物品小于承重，说明物品会放到背包。提前多赋值一次也是允许的
      dp[i + 1][v] = dp[i][v];
      if (v >= cost[i]) {
        dp[i + 1][v] = MAX(dp[i][v], dp[i][v - cost[i]] + value[i]);
      }
    }
  }

  for (int i = 0; i <= N; i++) {
    for (int j = 0; j <= V; j++) {
      printf("%d\t", dp[i][j]);
    }
    printf("\n");
  }
  printf("%d\n", dp[N][V]);
}

void O1Package_one_array_n() {

  int dp[V + 1];

  // 使用0个物品的价值为0
  for (int i = 0; i <= V; i++) {
    dp[i] = 0;
  }

  // 先遍历物品，后遍历重量
  for (int i = N - 1; i >= 0; i--) {
    for (int v = V; v >= cost[i]; v--) {
      int bufang = dp[v];
      int fang = dp[v - cost[i]] + value[i];
      dp[v] = MAX(bufang, fang);
    }
  }

  for (int j = 0; j <= V; j++) {
    printf("%d\t", dp[j]);
  }
  printf("\n");
  printf("%d\n", dp[V]);
}

```

## 完全背包

要求物品可以取无限个，但由于包的承重为V,所以每种物品最大数量为 V/cost[i]
```c
void CompletePackage_use_n() {

  int dp[N][V + 1];

  // 0容量的价值为0, 合法
  for (int i = 0; i < cost[0]; i++) {
    dp[0][i] = 0;
  }

  // 初始化当包承重大于物品0重量的情况
  for (int i = cost[0]; i <= V; i++) {
    // 当只装物品0时，包的价值就是物品0的价值
    dp[0][i] = i / cost[0] * value[0];
  }

  // 先遍历物品，后遍历重量
  for (int i = 1; i < N; i++) {
    for (int v = 0; v <= V; v++) {
      dp[i][v] = dp[i - 1][v]; //当重量不够装i时，价值为装i-1的价值,
                               //此步骤以备i+1时使用（下一个i循环）
      for (int k = 1; k <= v / cost[i]; k++) {
        dp[i][v] = MAX(dp[i - 1][v], dp[i - 1][v - k * cost[i]] + k * value[i]);
      }
    }
  }

  for (int i = 0; i < N; i++) {
    for (int j = 0; j <= V; j++) {
      printf("%d\t", dp[i][j]);
    }
    printf("\n");
  }
  printf("%d\n", dp[N - 1][V]);
}

void CompletePackage_use_n_plus_1() {

  int dp[N + 1][V + 1];

  // 0容量的价值为0, 合法
  for (int i = 0; i <= V; i++) {
    dp[0][i] = 0;
  }

  // 先遍历物品，后遍历重量
  for (int i = 1; i <= N; i++) {
    for (int v = 0; v <= V; v++) {
      dp[i][v] = dp[i - 1][v]; //当重量不够装i时，价值为装i-1的价值,
                               //此步骤以备i+1时使用（下一个i循环）
      for (int k = 1; k <= v / cost[i-1]; k++) {
        dp[i][v] = MAX(dp[i - 1][v], dp[i - 1][v - k * cost[i-1]] + k * value[i-1]);
      }
    }
  }

  for (int i = 0; i <= N; i++) {
    for (int j = 0; j <= V; j++) {
      printf("%d\t", dp[i][j]);
    }
    printf("\n");
  }
  printf("%d\n", dp[N][V]);
}

void CompletePackage_use_one_array_no_reverse_V() {

  int dp[V + 1];

  // 0容量的价值为0, 合法
  for (int i = 0; i <= V; i++) {
    dp[i] = 0;
  }

  // 先遍历物品，后遍历重量
  for (int i = 1; i <= N; i++) {
    for (int v = 0; v <= V; v++) {
      for (int k = 1; k <= v / cost[i-1]; k++) {
        dp[v] = MAX(dp[v], dp[v - k * cost[i-1]] + k * value[i-1]);
      }
    }
  }

  for (int j = 0; j <= V; j++) {
    printf("%d\t", dp[j]);
  }
  printf("\n");
  printf("%d\n", dp[V]);
}

void CompletePackage_use_one_array_reverse_V() {

  int dp[V + 1];

  // 0容量的价值为0, 合法
  for (int i = 0; i <= V; i++) {
    dp[i] = 0;
  }

  // 先遍历物品，后遍历重量
  for (int i = 1; i <= N; i++) {
    for (int v = V; v <= cost[i-1]; v--) {
      for (int k = 1; k <= v / cost[i-1]; k++) {
        dp[v] = MAX(dp[v], dp[v - k * cost[i-1]] + k * value[i-1]);
      }
    }
  }

  for (int j = 0; j <= V; j++) {
    printf("%d\t", dp[j]);
  }
  printf("\n");
  printf("%d\n", dp[V]);
}

```
## 多重背包

多重背包规定了每个物品的最大数量cap[i], 但由于要考虑背包承重，所以真正可以容纳物品的最大个数是max（cap[i], V/cost[i])

## 疑问

### 为何不需要将背包排序， 比如按重量排序或按价值排序，再将重量下而价值高的物品优先计算？

即使没有排序，由于第二层循环都是从`0->V`遍历和放物品i与不放物品i比较，所以顺序不重要
```c
  for (int i = 1; i < N; i++) {
    for (int v = 0; v <= V; v++) {
      int bufang = dp[i - 1][v];
      dp[i][v] = bufang;
      if (v >= cost[i]) {
        int fang = dp[i - 1][v - cost[i]] + value[i];
        dp[i][v] = MAX(bufang, fang);
      }
    }
  }
```

### 算法复杂度和优化

矩阵数组的时间复杂度为O(V*N),即两层for循环；空间复杂度为O(V*N) 即申请一个矩形数组，经过一元数组优化后，空间复杂度变为O(V)
除此之外，
大多情况下，通过两层遍历，可以只保留重量相同且价值最大的那个物品，这个操作的时间复杂度为O(N^2), 但当最糟糕的情况，即精心设计的物品没有相同重量或者相同价值的情况则此优化无效
另外，先加物品重量大于背包重量的物品忽略，可以加快速度

### 理解优化

无论是用矩阵数组还是优化成一元数组，最好的理解方式是单步调试一遍

### N行与N+1行

两种方式可以通用，只是当N行时，需要提前初始化物品0的情况

### 初始化问题

我们看到的求最优解的背包问题题目中，事实上有两种不太相同的问法。有的题目要求“恰好装满背包”时的最优解，有的题目则并没有要求必须把背包装满。一种区别这两种问法的实现方法是在初始化的时候有所不同。

如果是第一种问法，要求恰好装满背包，那么在初始化时除了 F[0] 为 0，其它F[1..V ] 均设为 −∞，这样就可以保证最终得到的 F[V ] 是一种恰好装满背包的最优解。
如果并没有要求必须把背包装满，而是只希望价格尽量大，初始化时应该将 F[0..V ]全部设为 0。

这是为什么呢？可以这样理解：初始化的 F 数组事实上就是在没有任何物品可以放入背包时的合法状态。如果要求背包恰好装满，那么此时只有容量为 0 的背包可以在什么也不装且价值为 0 的情况下被“恰好装满”，其它容量的背包均没有合法的解，属于未定义的状态，应该被赋值为 -∞ 了。如果背包并非必须被装满，那么任何容量的背包都有一个合法解“什么都不装”，这个解的价值为 0，所以初始时状态的值也就全部为 0了。

## 贪心算法

由于背包问题里的物品具备**重量**和**价值**两个维度，所以无法用贪心算法解决。而当维度为1的时候，可以用贪心算法，例如~~零钱问题~~和会议安排问题，因为不同零钱价值不同但其“重量”都为1，会议的“重量”也为1
**注**: leetcode上的[零钱问题](https://leetcode.com/problems/coin-change/)不能用贪心算法
## 参考
《背包问题九讲》
