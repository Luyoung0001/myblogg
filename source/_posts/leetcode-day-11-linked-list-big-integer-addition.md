---
title: "LeetCode 专题训练 Day 11：链表高精度加法"
date: 2023-03-23 12:55:29
tags:
  - "LeetCode"
  - "linked list"
  - "big integer"
categories:
  - "Algorithms"
---

<!--more-->

## 题目 1：[两数相加](https://leetcode.cn/problems/add-two-numbers/)

> 给你两个 非空 的链表，表示两个非负的整数。它们每位数字都是按照 逆序 的方式存储的，并且每个节点只能存储 一位 数字。
请你将两个数相加，并以相同形式返回一个表示和的链表。
你可以假设除了数字 0 之外，这两个数都不会以 0 开头。

示例 1：

> 输入：l1 = [2,4,3], l2 = [5,6,4]
输出：[7,0,8]
解释：342 + 465 = 807.

思路：模拟：

```java
 */
class Solution {
//    输入：l1 = [2,4,3], l2 = [5,6,4]
//    输出：[7,0,8]
//    解释：342 + 465 = 807.
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        int num1 = 0;
        int num2 = 0;
        int num = 0;
        int tag = 0;
        ListNode head = new ListNode();
        ListNode p = new ListNode();
        p.next = head;
        while(l1 != null || l2 != null){
            if(l1 != null && l2 != null){
                num1 = l1.val;
                num2 = l2.val;
                l1 = l1.next;
                l2 = l2.next;
            }else if(l1 == null){
                num1 = 0;
                num2 = l2.val;
                l2 = l2.next;
            }else if(l2 == null){
                num2  =0;
                num1 = l1.val;
                l1 = l1.next;
            }
            // 模拟
            num = (num1+num2+tag)%10;
            tag = (num1+num2+tag)/10;

            ListNode temp = new ListNode();
            temp.val = num;
            temp.next = null;
            p.next.next = temp;
            p = p.next;

        }
        if(tag != 0){
            ListNode temp = new ListNode();
            temp.val = tag;
            temp.next = null;
            p.next.next = temp;
            p = p.next;

        }
        return head.next;
    }
}
```

## 题目 2：[两数相加 II](https://leetcode.cn/problems/add-two-numbers-ii/)

> 给你两个 非空 链表来代表两个非负整数。数字最高位位于链表开始位置。它们的每个节点只存储一位数字。将这两数相加会返回一个新的链表。
你可以假设除了数字 0 之外，这两个数字都不会以零开头。
>
示例 1：
> 输入：l1 = [7,2,4,3], l2 = [5,6,4]
输出：[7,8,0,7]

思路 1：可以看到，必须从末尾进行相加，因此首先得对两个链表进行翻转，然后直接按照上一题的解法进行模拟：

```java
class Solution {
    public ListNode addTwoNumbers(ListNode ll1, ListNode ll2) {
        // 反转
        ListNode l1  = reverse(ll1);
        ListNode l2 = reverse(ll2);
        int num1 = 0;
        int num2 = 0;
        int num = 0;
        int tag = 0;
        ListNode head = new ListNode();
        ListNode p = new ListNode();
        p.next = head;
        while(l1 != null || l2 != null){
            if(l1 != null && l2 != null){
                num1 = l1.val;
                num2 = l2.val;
                l1 = l1.next;
                l2 = l2.next;
            }else if(l1 == null){
                num1 = 0;
                num2 = l2.val;
                l2 = l2.next;
            }else if(l2 == null){
                num2  =0;
                num1 = l1.val;
                l1 = l1.next;
            }
            // 模拟
            num = (num1+num2+tag)%10;
            tag = (num1+num2+tag)/10;

            ListNode temp = new ListNode();
            temp.val = num;
            temp.next = null;
            p.next.next = temp;
            p = p.next;

        }
        if(tag != 0){
            ListNode temp = new ListNode();
            temp.val = tag;
            temp.next = null;
            p.next.next = temp;
            p = p.next;
        }
        return reverse(head.next);

    }
    public ListNode reverse(ListNode l1){
        if(l1 == null || l1.next == null){
            return l1;
        }
        ListNode newnode = new ListNode();
        newnode = reverse(l1.next);
        l1.next.next = l1;
        l1.next = null;
        return newnode;
    }
}
```

## 题目 3：[面试题 02.05. 链表求和](https://leetcode.cn/problems/sum-lists-lcci/)

> 给定两个用链表表示的整数，每个节点包含一个数位。
这些数位是反向存放的，也就是个位排在链表首部。
编写函数对这两个整数求和，并用链表形式返回结果。
如果反向呢？

示例 1：

> 输入：(7 -> 1 -> 6) + (5 -> 9 -> 2)，即617 + 295
输出：2 -> 1 -> 9，即912

思路：这道题和第一道题一模一样，第二问和第二题一样，翻转一下就好：

```java
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        int num1 = 0;
        int num2 = 0;
        int num = 0;
        int tag = 0;
        ListNode head = new ListNode();
        ListNode p = new ListNode();
        p.next = head;
        while(l1 != null || l2 != null){
            if(l1 != null && l2 != null){
                num1 = l1.val;
                num2 = l2.val;
                l1 = l1.next;
                l2 = l2.next;
            }else if(l1 == null){
                num1 = 0;
                num2 = l2.val;
                l2 = l2.next;
            }else if(l2 == null){
                num2  =0;
                num1 = l1.val;
                l1 = l1.next;
            }
            // 模拟
            num = (num1+num2+tag)%10;
            tag = (num1+num2+tag)/10;

            ListNode temp = new ListNode();
            temp.val = num;
            temp.next = null;
            p.next.next = temp;
            p = p.next;

        }
        if(tag != 0){
            ListNode temp = new ListNode();
            temp.val = tag;
            temp.next = null;
            p.next.next = temp;
            p = p.next;

        }
        return head.next;
    }
}
```





