---
title: "LeetCode 刷题日志：2024-06-19"
date: 2024-06-19 21:11:43
tags:
  - "LeetCode"
  - "algorithms"
  - "data structures"
categories:
  - "Algorithms"
---

<!--more-->

## 41. 缺失的第一个正数
这个题目的通过率很低，是一道难题，类似于脑筋急转弯，确实不好想。**实际上，对于一个长度为 N 的数组，其中没有出现的最小正整数只能在 [1,N+1] 中。**

这个结论并不好想，举个例子：`nums = [3,4,-1,1]`，长度为 4，未出现的最小正数是 2；极端一点，`nums = [1,2,3,4]`，未出现的最小正数是 5。因此算法的第一步就是预处理，将这个范围之外的数全部标记掉：

```cpp
		for (int& num : nums) {
            if (num <= 0 || num > n) {
                num = n + 1;
            }
        }
```
对于 `nums = [3,4,-1,1]`，操作之后，`nums = [3,4,5,1]`。

第二步，需要将符合条件的数，放到它标记到应该呆的位置上，标记方法是取反，比如 1，就放到第一个位置 0，将每一个数操作一遍：

```cpp
		for (int i = 0; i < n; ++i) {
            int pos = abs(nums[i]) - 1; // 计算原始数字对应的索引
            if (pos < n && nums[pos] > 0) { // 只有在正常范围内的数才进行放置
                nums[pos] =
                    -nums[pos]; // 通过取负值来标记这个位置的数字已经存在
            }
        }

```

对于 `nums = [3,4,5,1]`，操作之后，`nums = [3,4,-5,1]`、`nums = [3,4,-5,-1]`、`nums = [3,4,-5,-1]`（不变）、`nums = [-3,4,-5,-1]`。经过这轮操作之后，会发现，第二个数没有被标记，因此数组中没有 2，因此将 2 返回：

```cpp
		for (int i = 0; i < n; ++i) {
            if (nums[i] > 0) {
                return i + 1; // 返回缺失的第一个正数
            }
        }
        return n + 1; // 如果1到n都存在，那么返回n+1
```

## 73. 矩阵置零

这道题目的思路是，首先判断第一行或者第一列是否有 0 元素：

```cpp
		bool firstRowZero = false;
        bool firstColZero = false;

        for (int i = 0; i < m; i++) {
            if (matrix[i][0] == 0) {
                firstColZero = true;
                break;
            }
        }

        for (int j = 0; j < n; j++) {
            if (matrix[0][j] == 0) {
                firstRowZero = true;
                break;
            }
        }
```

然后根据非第一列非第一行得元素是否为零，标记第一列或者第一行：

```cpp
 		for (int i = 1; i < m; i++) {
            for (int j = 1; j < n; j++) {
                if (matrix[i][j] == 0) {
                    matrix[i][0] = 0;
                    matrix[0][j] = 0;
                }
            }
        }
```
当第一行或者第一列被标记了，就可以根据这个信息来标记其它元素了：
```cpp
		for (int i = 1; i < m; i++) {
            for (int j = 1; j < n; j++) {
                if (matrix[i][0] == 0 || matrix[0][j] == 0) {
                    matrix[i][j] = 0;
                }
            }
        }
```

最后根据flags 置第一行第一列元素为 0：

```cpp
		if (firstColZero) {
            for (int i = 0; i < m; i++) {
                matrix[i][0] = 0;
            }
        }

        if (firstRowZero) {
            for (int j = 0; j < n; j++) {
                matrix[0][j] = 0;
            }
        }
```

## 48. 旋转图像
这个题目的思路是先转置，然后反转每一行：

```cpp
class Solution {
public:
    void rotate(vector<vector<int>>& matrix) {
        int n = matrix.size();
        // 转置矩阵
        for (int i = 0; i < n; ++i) {
            for (int j = i; j < n; ++j) {
                swap(matrix[i][j], matrix[j][i]);
            }
        }
        // 反转每一行
        for (int i = 0; i < n; ++i) {
            reverse(matrix[i].begin(), matrix[i].end());
        }
    }
};
```

## 240. 搜索二维矩阵 II
我以为这道题要从左上角开始搜索，后来才发现必须通过右上或者坐下，因为这两个方向中，元素单调性是相反的，单调性如果相同，是没办法排除的：

```cpp
class Solution {
public:
    bool searchMatrix(vector<vector<int>>& matrix, int target) {
        if (matrix.empty() || matrix[0].empty())
            return false;
        int m = matrix.size();
        int n = matrix[0].size();
        int i = 0, j = n - 1;

        while (i < m && j >= 0) {
            if (matrix[i][j] == target) {
                return true;
            } else if (matrix[i][j] > target) {
                j--;
            } else {
                i++;
            }
        }
        return false;
    }
};
```
## 160. 相交链表
这道题目简单，直接双指针，因为 A+B = B+A，它们这样运行一圈后，必然会相交：

```cpp
class Solution {
public:
    ListNode* getIntersectionNode(ListNode* headA, ListNode* headB) {
        if (headA == NULL || headB == NULL)
            return NULL;

        ListNode* pA = headA;
        ListNode* pB = headB;

        while (pA != pB) {
            // 如果达到末尾，则转向另一链表的头部
            pA = pA == NULL ? headB : pA->next;
            pB = pB == NULL ? headA : pB->next;
        }

        // 若相遇，则 pA 或 pB 为交点，或两者均为 NULL，未相交
        return pA;
    }
};
```

## 206. 反转链表
这个题目用两种解法，递归和迭代：

```cpp
class Solution {
public:
    ListNode* reverseList(ListNode* head) {
        // 递归
        // if(head ==  nullptr || head->next == nullptr)
        // return head;

        // ListNode *newNode = reverseList(head->next);
        // head->next->next = head;
        // head->next = nullptr;
        // return newNode;

        // 迭代

        ListNode* prev = nullptr; // 前一个结点
        ListNode* curr = head;    // 遍历
        ListNode* next = nullptr; // 存储下一个结点

        while (curr != nullptr) {
            next = curr->next; // 存储下一个结点
            curr->next = prev; // 反转当前
            prev = curr;       // 移动
            curr = next;       // 移动
        }
        return prev;
    }
};
```

## 234. 回文链表
这个题目能用到上面的解法，先用快慢指针找到中点，然后反转后面的链表，然后双指针从两边向中心靠近且比较：

```cpp
class Solution {
public:
    bool isPalindrome(ListNode* head) {
        if (head == nullptr || head->next == nullptr)
            return true;

        ListNode *slow = head, *fast = head, *prev = nullptr, *next = nullptr;


        while (fast != nullptr && fast->next != nullptr) {
            fast = fast->next->next;
            slow = slow->next;
        }

        while (slow != nullptr) {
            next = slow->next;
            slow->next = prev;
            prev = slow;
            slow = next;
        }

        ListNode* left = head;
        ListNode* right = prev;
        while (right != nullptr) {
            if (left->val != right->val)
                return false;
            left = left->next;
            right = right->next;
        }

        return true;
    }
};
```

## 141. 环形链表

快慢指针：

```cpp
	bool hasCycle(ListNode* head) {
        ListNode *fast = head, *slow = head;
        while (fast != nullptr && slow != nullptr) {
            fast = fast->next;
            if (fast == nullptr)
                return false;
            fast = fast->next;
            slow = slow->next;
            if (slow == fast)
                return true;
        }
        return false;
    }
```

## 142. 环形链表 II
这个用哈希表简直是降维打击：

```cpp
	ListNode *detectCycle(ListNode *head) {
        unordered_set<ListNode *> visited;
        while (head != nullptr) {
            if (visited.count(head)) {
                return head;
            }
            visited.insert(head);
            head = head->next;
        }
        return nullptr;
    }
```
## 21. 合并两个有序链表
使用一个伪头结点，然后将所有的结点挂在这个结点上：

```cpp
class Solution {
public:
    ListNode* mergeTwoLists(ListNode* list1, ListNode* list2) {
        ListNode dummy(0);
        ListNode* tail = &dummy; // 使用一个尾指针，始终指向新链表的最后一个节点

        while (list1 && list2) {
            if (list1->val < list2->val) {
                tail->next = list1;  // 接上 list1 的当前节点
                list1 = list1->next; // 移动 list1 指针
            } else {
                tail->next = list2;  // 接上 list2 的当前节点
                list2 = list2->next; // 移动 list2 指针
            }
            tail = tail->next; // 移动尾指针到新链表的末尾
        }

        tail->next = list1 ? list1 : list2;

        return dummy.next;
    }
};
```
## 总结
### 41. 缺失的第一个正数
- 利用索引作为哈希键通过元素取反来标记存在的数字，有效利用数组自身空间来存储信息。
- 确保处理后的数组中，每个数字都在其应有的位置上，没有则是缺失的最小正数。

### 73. 矩阵置零
- 使用矩阵的第一行和第一列作为标记数组，记录哪些行列需要被置零。
- 分步处理，先标记，后置零，注意操作的顺序和逻辑清晰。

### 48. 旋转图像
- 先转置矩阵，然后反转每一行，是一种简洁的在原地操作矩阵的方法，符合题目要求的空间复杂度。

### 240. 搜索二维矩阵 II
- 从右上角（或左下角）开始搜索，利用行和列的单调性来排除行或列，这种策略提高了搜索效率。

### 160. 相交链表
- 双指针法，一个从链表A出发到链表B，另一个从链表B出发到链表A，两者会在交点相遇。
- 由于路径长度相同（A+B=B+A），所以两指针会同时到达交点。

### 206. 反转链表
- 迭代和递归两种方式，迭代方式通过交换指针方向在原地反转链表，而递归则是通过递归栈来实现。

### 234. 回文链表
- 快慢指针找到中点，反转后半部分，然后前后两部分进行比对。
- 这种方法充分利用了链表的特性，减少了空间复杂度。

### 141. 环形链表
- 快慢指针法检测环，快指针每次走两步，慢指针每次走一步，如果相遇则说明有环。

### 142. 环形链表 II
- 使用哈希表记录访问过的节点，第一个重复访问的节点即为环的入口。
- 或者使用快慢指针确定环的存在后，用两个指针从头和相遇点出发，第二次相遇点即为环的入口。

### 21. 合并两个有序链表
- 使用一个伪头节点简化边界情况的处理，通过比较两链表头部节点的值，依次选择较小的节点拼接到新链表上。



