---
title: "C 链表实现：头插法与尾插法"
date: 2023-02-28 18:54:15
tags:
  - "C"
  - "linked list"
  - "data structure"
categories:
  - "Programming Languages"
---

<!--more-->

## 前言
如果想在链表的首位置增加结点，就是在头结点后一个结点插入一个结点，把 p 指针指向头结点就可以操作了；如果要在末尾增加结点，那么指针 p 必须指向最后一个结点，然后也就可以开始操作了。
## 代码

```c
#include <stdio.h>
#include <stdlib.h>

// 随便定义一个链表节点
typedef struct node {
    char* name;
    int age;
    struct node* next;

} LinkList;

int main() {
    // 初始化一个链表
    LinkList* HeadNode = (LinkList*)malloc(sizeof(LinkList));
    // 初始化头结点,我们不用它,随便初始化个值就好
    (*HeadNode).age = 0;
    (*HeadNode).name = NULL;
    // 由于连链表此时只有一个结点,我先用头插法,就是头结点之后插

    // p指向头结点
    LinkList* p = HeadNode;

    // 创造一个新节点,并且 指针node指向它
    LinkList* node = (LinkList*)malloc(sizeof(LinkList));
    // 简单赋值
    (*node).age = 10;
    (*node).name = "张三";

    // 开始插入
    p->next = node;
    node->next = NULL;
    // 插入完毕

    // 再次插入

    // 创造一个新节点,并且 指针node1指向它
    LinkList* node1 = (LinkList*)malloc(sizeof(LinkList));
    // 简单赋值
    (*node1).age = 11;
    (*node1).name = "李四";

    // 开始插入
    node1->next = p->next;
    p->next = node1;

    // 插入完毕

    // 这样,链表中就有 1 个头结点和2 个节点了
    // 我们遍历看看
    // 遍历也要重新声明一个指针,指向头结点后的第一个结点
    LinkList* q = HeadNode->next;
    while (q) {
        printf("age:%d name:%s\n", q->age, q->name);
        q = q->next;
    }
    // 打印的结果应该是:李四,张三
    // 接下来,进行尾插
    // 用一个指针,指向最后一个元素
    LinkList* t = HeadNode;
    // 当 t->next为空时,退出循环,此时,t 指向最后一个节点
    while (t->next) {
        t = t->next;
    }
    // 新建一个结点
    LinkList* node3 = (LinkList*)malloc(sizeof(LinkList));
    (*node3).age = 12;
    (*node3).name = "王五";

    // 后端插入
    t->next = node3;
    node3->next = NULL;
    // 就这么简单
    // 再来遍历一次
    LinkList* q1 = HeadNode->next;
    printf("第二次遍历:\n");
    while (q1) {
        printf("age:%d name:%s\n", q1->age, q1->name);
        q1 = q1->next;
    }
    // 打印的结果应该是:李四,张三,王五

    return 0;
}
```
## 输出结果

> age:11 name:李四
age:10 name:张三
第二次遍历:
age:11 name:李四
age:10 name:张三
age:12 name:王五

可以看到，结果完全符合我们的预期。


*全文完，感谢阅读。*

