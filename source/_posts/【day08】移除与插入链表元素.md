---
title: 【day08】移除与插入链表元素
date: 2023-03-20 22:37:14
tags: 链表 数据结构 算法
categories: LeetCode
---

<!--more-->

## 题目 1：[设计链表](https://leetcode.cn/problems/design-linked-list/)

> 你可以选择使用单链表或者双链表，设计并实现自己的链表。
单链表中的节点应该具备两个属性：val 和 next 。val 是当前节点的值，next 是指向下一个节点的指针/引用。
如果是双向链表，则还需要属性 prev 以指示链表中的上一个节点。假设链表中的所有节点下标从 0 开始。
实现 MyLinkedList 类：
MyLinkedList() 初始化 MyLinkedList 对象。
int get(int index) 获取链表中下标为 index 的节点的值。如果下标无效，则返回 -1 。
void addAtHead(int val) 将一个值为 val 的节点插入到链表中第一个元素之前。在插入完成后，新节点会成为链表的第一个节点。
void addAtTail(int val) 将一个值为 val 的节点追加到链表中作为链表的最后一个元素。
void addAtIndex(int index, int val) 将一个值为 val 的节点插入到链表中下标为 index 的节点之前。如果 index 等于链表的长度，那么该节点会被追加到链表的末尾。如果 index 比长度更大，该节点将 不会插入 到链表中。
void deleteAtIndex(int index) 如果下标有效，则删除链表中下标为 index 的节点。

思路：

```c
typedef struct MyLinkedList_t {
    int val;
    struct MyLinkedList_t *next;
} MyLinkedList;



MyLinkedList* myLinkedListCreate() {
 MyLinkedList *obj = (MyLinkedList *)malloc(sizeof(MyLinkedList));
	obj->val = 0;
	obj->next = NULL;
    return obj;

 
}

int myLinkedListGet(MyLinkedList* obj, int index) {

    int target;
    MyLinkedList* ob = obj;
        while (index--&&ob->next!=NULL)
        {
            ob = ob->next;
    }
    if(ob->next!=NULL)
        return target = ob ->next->val;
     return -1;
}
// 这是有头结点的链表
void myLinkedListAddAtHead(MyLinkedList* obj, int val) {
MyLinkedList* p = (MyLinkedList*)malloc(sizeof(MyLinkedList));
        
        p->val = val;
        p->next = obj->next;
        obj->next = p;
}

void myLinkedListAddAtTail(MyLinkedList* obj, int val) {
MyLinkedList* cur = obj;
     
    while (cur->next!=NULL)
    {
        cur=cur->next;
    }
       MyLinkedList* p = (MyLinkedList*)malloc(sizeof(MyLinkedList));
        p->next = NULL;
        p->val = val;
        cur->next = p;
    
}

void myLinkedListAddAtIndex(MyLinkedList* obj, int index, int val) {
 
    if (index <= 0)
    {
        MyLinkedList* p = (MyLinkedList*)malloc(sizeof(MyLinkedList));
        p->next = NULL;
        p->val = val;
        p->next = obj->next;
        obj->next=p;
        return;
    }
    MyLinkedList* ob = obj;
    while (index--&& ob != NULL)
    {
        ob= ob->next;
    }
    if (ob == NULL)
        return;
    MyLinkedList* p = (MyLinkedList*)malloc(sizeof(MyLinkedList));
    p->next = ob->next;
    p->val= val;
    ob->next=p;

}

void myLinkedListDeleteAtIndex(MyLinkedList* obj, int index) {


    if (index < 0||obj->next == NULL)
        return;
    MyLinkedList*  ob = obj;
    MyLinkedList*  cur;
    while (index-- && ob->next != NULL)
    {
        ob = ob->next;
    }
    if(ob->next==NULL)
        return ;
    cur= ob->next;
    ob->next = cur->next;
    free(cur);

}

void myNodeFree(MyLinkedList* Node) {
	if (Node->next != NULL) {
		myNodeFree(Node->next);
		//Node->next = NULL;
	}
	free(Node);
}
void myLinkedListFree(MyLinkedList* obj) {
    myNodeFree(obj);
}



/**
 * Your MyLinkedList struct will be instantiated and called as such:
 * MyLinkedList* obj = myLinkedListCreate();
 * int param_1 = myLinkedListGet(obj, index);
 
 * myLinkedListAddAtHead(obj, val);
 
 * myLinkedListAddAtTail(obj, val);
 
 * myLinkedListAddAtIndex(obj, index, val);
 
 * myLinkedListDeleteAtIndex(obj, index);
 
 * myLinkedListFree(obj);
*/
```

## 题目 2：[移除链表元素](https://leetcode.cn/problems/remove-linked-list-elements/)

> 给你一个链表的头节点 head 和一个整数 val ，请你删除链表中所有满足 Node.val == val 的节点，并返回 新的头节点 。

示例 1：

> 输入：head = [1,2,6,3,4,5,6], val = 6
输出：[1,2,3,4,5]

思路：递归

```java
class Solution {
    public ListNode removeElements(ListNode head, int val) {
       if(head==null)
           return null;
        head.next=removeElements(head.next,val);
        if(head.val==val){
            return head.next;
        }else{
            return head;
        }
    }
}
```
## 题目 3： [删除链表中的节点](https://leetcode.cn/problems/delete-node-in-a-linked-list/)

> 有一个单链表的 head，我们想删除它其中的一个节点 node。
给你一个需要删除的节点 node 。你将 无法访问 第一个节点  head。
链表的所有值都是 唯一的，并且保证给定的节点 node 不是链表中的最后一个节点。


示例 1：

> 输入：head = [4,5,1,9], node = 5
输出：[4,1,9]
解释：指定链表中值为 5 的第二个节点，那么在调用了你的函数之后，该链表应变为 4 -> 1 -> 9

思路：

```java
class Solution {
    public void deleteNode(ListNode node) {
        ListNode p = node;
        ListNode q = p.next;
        while(q.next != null){
            p.val = q.val;
            p = p.next;
            q = q.next;
        }
        p.val = q.val;
        p.next = null;
        
    }
}
```

## 题目 4：[删除链表的倒数第 N 个结点](https://leetcode.cn/problems/remove-nth-node-from-end-of-list/)
> 给你一个链表，删除链表的倒数第 n 个结点，并且返回链表的头结点。

示例 1：

> 输入：head = [1,2,3,4,5], n = 2
输出：[1,2,3,5]

思路：

```java
class Solution {
    public ListNode removeNthFromEnd(ListNode head, int n) {
        // 计算长度
        int len = 0;
        ListNode t = head;
        while(t != null){
            ++len;
            t = t.next;
        }
        // 设置头结点，可以删除头结点
        ListNode d = new ListNode(0, head);
        ListNode p = d;
        // 指向删除结点前的一个
        // 输入：head = [1,2,3,4,5], n = 2
        // 输出：[1,2,3,5]
        for(int i = 0; i < len-n; i++){ 
            p = p.next;
        }
        // 删除结点
        p.next = p.next.next;
        return d.next;


    }
}
```

## 题目 5：[删除排序链表中的重复元素](https://leetcode.cn/problems/remove-duplicates-from-sorted-list/)

> 给定一个已排序的链表的头 head ， 删除所有重复的元素，使每个元素只出现一次 。返回 已排序的链表 。

示例 1：

> 输入：head = [1,1,2]
输出：[1,2]


思路 1：

```java
class Solution {
    public ListNode deleteDuplicates(ListNode head) {
        if(head == null) return null;
        ListNode p = head;
        // 1,1,1,2,2,3,3
        while(p.next!= null){
            while(p.val == p.next.val){
                p.next = p.next.next;
                if(p.next == null) return head;
            }
            p = p.next;
        }
        return head;
    }
}
```


## 题目 6：[删除排序链表中的重复元素 II](https://leetcode.cn/problems/remove-duplicates-from-sorted-list-ii/)

> 给定一个已排序的链表的头 head ， 删除原始链表中所有重复数字的节点，只留下不同的数字 。返回 已排序的链表 。

示例 1：
> 输入：head = [1,2,3,3,4,4,5]
输出：[1,2,5]

思路 1：

```c
struct ListNode* deleteDuplicates(struct ListNode* head){
    if(head == NULL){
        return head;
    }
    if(head->next != NULL && head->val == head->next->val){
        while(head->next != NULL && head->val == head->next->val){
            head = head->next;//删除最后一个相同的元素之前所有相同的元素
        }
        return deleteDuplicates(head->next);//删除最后一个相同的元素
    }
    head->next = deleteDuplicates(head->next);
    return head;
    
}
```




