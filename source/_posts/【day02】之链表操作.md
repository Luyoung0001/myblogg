---
title: 【day02】之链表操作
date: 2023-03-08 22:41:48
tags: 链表 数据结构 算法
categories: LeetCode
---

<!--more-->

## 题目 5：[反转链表](https://leetcode.cn/problems/reverse-linked-list/)

> 给你单链表的头节点 head ，请你反转链表，并返回反转后的链表。


示例 1：

> 输入：head = [1,2,3,4,5]
输出：[5,4,3,2,1]

思路很简单，直接从最后一个结点逐渐指向第一个，并每次返回指向的第一个节点：
### 答案 1：

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
如果使用迭代，就得额外设置两个结点：

```java

class Solution {
    public ListNode reverseList(ListNode head) {
        ListNode prev = null;
        ListNode curr = head;
        while (curr != null) {
            ListNode next = curr.next;
            curr.next = prev;
            prev = curr;
            curr = next;
        }
        return prev;
    }
}
```
## 题目 6：[合并两个有序链表](https://leetcode.cn/problems/merge-two-sorted-lists/)

> 将两个升序链表合并为一个新的 升序 链表并返回。新链表是通过拼接给定的两个链表的所有节点组成的。

思路很简单，因为链表的定义是递归的，因此它的很多算法都可以用递归来做，因此这道题最好也是使用递归：
### 答案 1：

```java
class Solution {
    public ListNode mergeTwoLists(ListNode list1, ListNode list2) {
//        输入：l1 = [1,2,4], l2 = [1,3,4]
//        输出：[1,1,2,3,4,4]
        while(list1 != null || list2 != null){
            if(list1 == null){
                return list2;
            }else if(list2 == null){
                return list1;
            }else if(list1.val < list2.val){
                list1.next = mergeTwoLists(list1.next, list2);
                return list1;
            }else {
                list2.next = mergeTwoLists(list1, list2.next);
                return list2;
            }
        }
        return null;
    }
}
```
## 题目 7：[排序链表](https://leetcode.cn/problems/sort-list/)

> 给你链表的头结点 head ，请将其按 升序 排列并返回 排序后的链表 。

这里采用归并排序：

```java
struct ListNode* merge(struct ListNode* head1, struct ListNode* head2) {
    if(!head1){
        return head2;
    }else if(!head2){
        return head1;
    }else if(head1->val < head2->val){
        head1->next = merge(head1->next, head2);
        return head1;
    }else{
        head2->next = merge(head1, head2->next);
        return head2;
    }
}

struct ListNode* toSortList(struct ListNode* head, struct ListNode* tail) {
    if(head == NULL || head->next == NULL){
        return head;
    }
    if(head->next == tail){
        head->next = NULL;
        return head;
    }
    struct ListNode *slow = head, *fast = head;
    // 寻找中点~
    while (fast != tail) {
        slow = slow->next;
        fast = fast->next;
        if (fast != tail) {
            fast = fast->next;
        }
    }
    struct ListNode* mid = slow;
    return merge(toSortList(head, mid), toSortList(mid, tail));
}

struct ListNode* sortList(struct ListNode* head) {
    return toSortList(head, NULL);
}
```







