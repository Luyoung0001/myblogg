---
title: 【day10】链表的合并
date: 2023-03-22 21:39:59
tags: 链表 数据结构 算法
categories: LeetCode
---

<!--more-->

## 题目 1：[合并两个有序链表](https://leetcode.cn/problems/merge-two-sorted-lists/)

> 将两个升序链表合并为一个新的 升序 链表并返回。新链表是通过拼接给定的两个链表的所有节点组成的。 

示例 1：

> 输入：l1 = [1,2,4], l2 = [1,3,4]
输出：[1,1,2,3,4,4]

思路：采用递归的方法：

```java
class Solution {
    public ListNode mergeTwoLists(ListNode list1, ListNode list2) {
        if(list1 == null) return list2;
        if(list2 == null) return list1;
        if(list1.val >= list2.val){
            list2.next = mergeTwoLists(list1,list2.next);
            return list2;
        }else{
            list1.next = mergeTwoLists(list1.next, list2);
            return list1;
        }
    }
}
```

## 题目 2：[合并 K 个升序链表](https://leetcode.cn/problems/merge-k-sorted-lists/)

> 给你一个链表数组，每个链表都已经按升序排列。
请你将所有链表合并到一个升序链表中，返回合并后的链表。

示例 1：

> 输入：lists = [[1,4,5],[1,3,4],[2,6]]
输出：[1,1,2,3,4,4,5,6]
解释：链表数组如下：
[
  1->4->5,
  1->3->4,
  2->6
]
将它们合并到一个有序链表中得到。
1->1->2->3->4->4->5->6

思路 1：采用迭代加递归的方法，将链表数组的链表逐一合并：

```java
class Solution {
    public ListNode mergeKLists(ListNode[] lists) {
        int len = lists.length;
        if(len == 0){
            return null;
        }else if(len == 1){
            return lists[0];
        }
        ListNode ans = lists[0];
        for (int i = 1; i < len; i++) {
            ans = mergeTwoLists(ans, lists[i]);
        }
        return ans;
    }

    public ListNode mergeTwoLists(ListNode list1, ListNode list2) {
        if (list1 == null) return list2;
        if (list2 == null) return list1;
        if (list1.val >= list2.val) {
            list2.next = mergeTwoLists(list1, list2.next);
            return list2;
        } else {
            list1.next = mergeTwoLists(list1.next, list2);
            return list1;
        }
    }
}
```
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/311647537b7c476aa6779cc46165f725_1720253831933.png?token=ANB4BCP53RKA4MVVIPLXKA3GRD64O)

由于这个算法的时间复杂度为 O(k^2*n)，因此运行时间不理想，下面为改进的算法(分治)：

```java
class Solution {
    public ListNode mergeKLists(ListNode[] lists) {
        return merge(lists, 0, lists.length - 1);
    }

    public ListNode merge(ListNode[] lists, int l, int r) {
        if (l == r) {
            return lists[l];
        }
        if (l > r) {
            return null;
        }
        int mid = (l + r) >> 1;
        return mergeTwoLists(merge(lists, l, mid), merge(lists, mid + 1, r));
    }

    public ListNode mergeTwoLists(ListNode a, ListNode b) {
        if (a == null || b == null) {
            return a != null ? a : b;
        }
        ListNode head = new ListNode(0);
        ListNode tail = head, aPtr = a, bPtr = b;
        while (aPtr != null && bPtr != null) {
            if (aPtr.val < bPtr.val) {
                tail.next = aPtr;
                aPtr = aPtr.next;
            } else {
                tail.next = bPtr;
                bPtr = bPtr.next;
            }
            tail = tail.next;
        }
        tail.next = (aPtr != null ? aPtr : bPtr);
        return head.next;
    }
}

```
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/5ca96b9d65b744f88bbe9ccfde4c3f7a_1720253831933.png?token=ANB4BCIZ6SXERJDOSOOAI5DGRD64S)
[优化成分治](https://leetcode.cn/problems/merge-k-sorted-lists/solution/he-bing-kge-pai-xu-lian-biao-by-leetcode-solutio-2/)，使得在遍历 lists 时的时间复杂度大大降低：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/19b14ee63aad48778bfab87bf2d9e681_1720253831934.png?token=ANB4BCNPAGHYCP3RXZOSMQTGRD64W)




