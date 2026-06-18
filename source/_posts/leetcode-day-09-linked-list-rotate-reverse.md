---
title: "LeetCode 专题训练 Day 09：链表旋转与反转"
date: 2023-03-21 23:18:45
tags:
  - "LeetCode"
  - "linked list"
categories:
  - "Algorithms"
---

<!--more-->

## 题目 1：[旋转链表](https://leetcode.cn/problems/rotate-list/)

> 给你一个链表的头节点 head ，旋转链表，将链表每个节点向右移动 k 个位置。

示例 1：

> 输入：head = [1,2,3,4,5], k = 2
输出：[4,5,1,2,3]

思路 1：每次写后退一个，然后循环 k 次：

```c
struct ListNode* rotateRight(struct ListNode* head, int k){
    if(head == NULL || head->next == NULL || k == 0){
        return head;
    }

    // 求长度，且将其弄成环
    struct ListNode *t = head;
    int len = 0;
    while(t != NULL){
        ++len;
        t = t->next;
        if(t->next == NULL){
            t->next = head;
            break;
        }
    }
    //可以发现，链表是在倒数第k+1个结点之后切断的
    //求出切断点
    int cutPosition = len - k%(len+1);
    struct ListNode *ret = head;
    for(int i = 1; i <= cutPosition; i++){
        ret = ret->next;
    }
    struct ListNode *retu = ret->next;
    ret->next = NULL;
    return retu;
}
```

## 题目2：[反转链表](https://leetcode.cn/problems/reverse-linked-list/)

> 给你单链表的头节点 head ，请你反转链表，并返回反转后的链表。

示例 1：

> 输入：head = [1,2,3,4,5]
输出：[5,4,3,2,1]

思路 1：直接递归：

```java
class Solution {
    public ListNode reverseList(ListNode head) {
        if(head == null || head.next == null) return head;
        ListNode newNode = new ListNode();
        newNode = reverseList(head.next);
        head.next.next = head;
        head.next = null;
        return newNode;
    }
}
```

