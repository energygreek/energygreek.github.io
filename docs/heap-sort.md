---
date: 2021-12-13
tags: [算法]
---

# 堆排序

堆排类似选择排序，只排序未排序的部分，但堆排比较方式不同，每次只比较log(n)次  
最终的堆排时间复杂度为O(nlogn)

## 关键下标

堆一般用数组来表示，但是关系为

* 当以1为起点时，若父节点是n
- 左子节点=n*2
- 右子节点=n*2 + 1

- 若节点为n, 则其父节点为n/2
- 若元素为m个，则最后一个根节点（非叶节点）为m/2

* 当以0为起点时，若父节点是n
- 左子节点=(n+1)*2-1 = n*2 + 1
- 右子节点=(n+1)*2+1 -1 = n*2 + 2

- 若节点下标为n, 其父节点为(n+1)/2 - 1 = (n-1)/2
- 若元素为m个，则最后一个根节点（非叶节点）为(m+1)/2-1 = (m-1)/2

可见，若以1为起点，要简洁很多

## 建立堆

1. 从第一个非叶子节点开始，比较父子节点中找的最值，如果需要调整则交换父子节点再递归遍历下一层：
* 建立最小堆时，每次比较出最大值，一遍之后，堆顶为最大
* 建立最大堆时，每次比较出最小值，一遍之后，堆顶为最小
2. 比较到顶部节点时，完成第一次遍历，找到了顶部的最值
3. 将堆顶的最值与尾部交换，遍历范围-1，然后从堆顶开始比较
4. 重复3动作，直到范围为1

3的动作类似`选择排序`，只遍历未排序的部分


## 最大堆和最小堆

最小堆可以求得topK**大**的数，反过来最大堆可以求得topK**小**的数

### topK

堆排序通常用来实现topK，适用于限制内存的情况。方法是建立一个堆大小为K： 先用头部K个数据建立最小堆，后续的数字与堆顶（最小）比，如果跟小则跳过，否则替换然后执行thiftdown。其时间复杂读为O)KlogK + NlogK), 只需要一次建堆，后续都是比较。空间复杂度为K。

也可以则使用分治法，将数据分为多组，每组求其top k。最后合并起来再排序，求最终的topK。 分治法可以配合线程一起使用加快速度。


```c
// 使用最小堆来实现topK
void thiftdown(int *heap, int root, int size) {
  int m = root;
  int l = root * 2 + 1;
  int r = root * 2 + 2;

  if (l < size && heap[l] > heap[m]) {
    m = l;
  }
  if (r < size && heap[r] > heap[m]) {
    m = r;
  }

  if (m != root) {
    int tmp = heap[root];
    heap[root] = heap[m];
    heap[m] = tmp;

    thiftdown(heap, m, size);
  }
}

void adjust(int *heap, int root, int size) {
  int m = root;
  int l = root * 2 + 1;
  int r = root * 2 + 2;

  if (l < size && heap[l] < heap[m]) {
    m = l;
  }
  if (r < size && heap[r] < heap[m]) {
    m = r;
  }

  if (m != root) {
    int tmp = heap[root];
    heap[root] = heap[m];
    heap[m] = tmp;

    adjust(heap, m, size);
  }
}

void heapify(int *heap, int size) {
  for (int i = (size - 1) / 2; i >= 0; i--) {
    thiftdown(heap, i, size);
  }

  while (size > 0) {
    int tmp = heap[0];
    heap[0] = heap[size - 1];
    heap[size - 1] = tmp;
    size--;
    thiftdown(heap, 0, size);
  }
}

void add(int *heap, int val, int size) {
  if (heap[0] >= val) {
    return;
  }
  heap[0] = val;
  adjust(heap, 0, size);
}

```

使用方法
```c
heapify(m, 5);
for (int i = 5; i < 49; i++) {
  add(m, n[i], 5);
}
return m[0];
```

### 海量数据求topK

常见问题类似：
1. 求许多重复整数中，重复第K多的正整数。
2. 求文本中重复出现的词语次数，重复第K多的”，
3. ip中访问最K多的”。

这些问题需要多一步统计，统计出现次数，然后再使用堆来查找topK。

1. 使用数组统计，如果目标都是正整数，如问题1
2. 使用map<target, int>统计，int表示出现次数，如果目标字符串，如问题2和问题3

统计完成后，将目标和计数放入结构体，再使用堆来找topK
```
{
  目标
  次数
}
```

## 总结

topK 问题除了堆排序以外，还常用快排来找。