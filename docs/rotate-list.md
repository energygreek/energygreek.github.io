---
date: 2022-04-21
tags: [算法]
---

# 单向链表翻转算法

## 三指针法翻转单向链表

定义3个指针, 原理是三个指针分别指向前三个元素，将第一个元素指向第二个元素，然后全部后移
1. 第三个指针向前移动
2. 将第二个元素指向第一个元素
3. 第一个指针和第二个指针向前移动

```c
struct Node{
　　int data;
　　Node *next;
};

Node *reverse(Node *head) {
  Node *pPrev = nullptr;
  Node *pHead = head;
  Node *pNext = nullptr;
  while (pHead) {
    pNext = pHead->next;
    pHead->next = pPrev;
    pPrev = pHead;
    pHead = pNext;
  }
  return pHead;
}

```

```c
Node *reverse(Node *head) {
  if (!head) {
    return head;
  }
  Node *first = nullptr;
  Node *second = head;
  Node *third = head->next;

  while (third) {
    second->next = first;

    first = second;
    second = third;
    third = third->next;
  }

  second->next = first;

  return second;
}
```

## 翻转链表的部分节点

翻转node[m]->node[N]部分， 理解时先将链表分为3部分: part1->part2->part3, 只翻转part2  

翻转动作套用三指针法，但部分翻转的难点在于处理part1为空和非空的情况，所以要判断m时候为1
* m==1即要翻转第一个元素时，需要先记录part2翻转后的tail node，最后将其next指向part3的第一个元素
* m!=1即part1不为空，需要先记录part2翻转后的tail node和part1的最后一个node，最后将其指向part2的新head,以及将part2的第一个node指向part3

```c
struct ListNode *reverseBetween(struct ListNode *head, int m, int n) {
  if (m == n) {
    return head;
  }

  if(!head || !head->next){
    return head;
  }

  // write code here
  struct ListNode *pleft = NULL;
  struct ListNode *phead = head;
  struct ListNode *pright = NULL;


  // 只有当m！=1时才需要，指向part1的最后一个元素
  struct ListNode *pPart1Tail = NULL;
  // 当m=1时指向part2的第一个元素
  struct ListNode *pPart2NewTail = NULL;

  // skip m
  for (int i = 1; i < m; i++) {
    pleft = phead; 
    phead = phead->next;
  }

  // 保存part2的head,也是翻转后的tail
  pPart2NewTail = phead;
  // m != 1 时pleft不为NULL
  struct ListNode *pPartOneTail = pleft;

  int i = 0;
  // 翻转次数要比m-n大1
  while (phead && i < n - m + 1) {
    pright = phead->next;

    phead->next = pleft;

    pleft = phead;
    phead = pright;
    i++;
  }

  if(m==1){
    // 指向part3的头
    pPart2NewTail->next = phead;
    return pleft;
  }

  pPartOneTail->next = pleft;
  pPart2NewTail->next = phead;

  return head;
}
```