---
title: "LeetCode 刷题日志：2024-06-30"
date: 2024-06-30 23:29:37
tags:
  - "LeetCode"
  - "algorithms"
  - "data structures"
categories:
  - "Algorithms"
---

<!--more-->

## 121. 买卖股票的最佳时机
实例 1：

> 输入：[7,1,5,3,6,4]
> 输出：5
> 解释：在第 2 天（股票价格 = 1）的时候买入，在第 5 天（股票价格 = 6）的时候卖出，最大利润 = 6-1 = 5 。注意利润不能是 7-1 = 6, 因为卖出价格需要大于买入价格；同时，你不能在买入前卖出股票。

示例 2：

> 输入：prices = [7,6,4,3,1]
> 输出：0
> 解释：在这种情况下, 没有交易完成, 所以最大利润为 0。

思路就是，一边遍历，一边更新 maxProfit，然后更新 历史最低价，并期望在未来能获得最大利润：

```cpp
class Solution {
public:
    int maxProfit(vector<int>& prices) {
        int min = prices[0], maxProfit = 0;
        for (int price : prices) {
            maxProfit = price > min && price - min > maxProfit ? price - min
                                                               : maxProfit;
            min = price < min ? price : min;
        }
        return maxProfit;
    }
};
```

## 55. 跳跃游戏
示例 1：

> 输入：nums = [2,3,1,1,4]
> 输出：true
> 解释：可以先跳 1 步，从下标 0 到达下标 1, 然后再从下标 1 跳 3
> 步到达最后一个下标。

示例 2：

> 输入：nums = [3,2,1,0,4]
> 输出：false
> 解释：无论怎样，总会到达下标为 3 的位置。但该下标的最大跳跃长度是 0，所以永远不可能到达最后一个下标。

思路是一边遍历，一边更新 **maxReach**：

```cpp
class Solution {
public:
    bool canJump(vector<int>& nums) {
        int maxReach = 0;    // 最远可以到达的位置
        int n = nums.size(); // 数组的长度

        for (int i = 0; i < n; i++) {
            if (i > maxReach) {
                return false;
            }
            // 更新最远可以到达的位置
            maxReach = max(maxReach, i + nums[i]);
            if (maxReach >= n - 1) {
                return true;
            }
        }

        return false;
    }
};

```

## 45. 跳跃游戏 II

示例 1:

> 输入: nums = [2,3,1,1,4]
> 输出: 2
> 解释: 跳到最后一个位置的最小跳跃数是 2。从下标为 0 跳到下标为 1 的位置，跳 1 步，然后跳 3 步到达数组的最后一个位置。

示例 2:

> 输入: nums = [2,3,0,1,4]
> 输出: 2

这道题目返回的是，最小的跳跃次数。应该思考，什么时候次数+1？答案是，当 i 遍历到currentEnd 的时候，就意味着还需要再跳一次。当然这个确定再跳一次后，可以提前结束。

```cpp
class Solution {
public:
    int jump(vector<int>& nums) {
        int len = nums.size();
        if (len == 1) return 0;
        int maxReach = 0, times = 0;
        int currentEnd = 0;

        for (int i = 0; i < len; i++) {
            maxReach = max(maxReach, i + nums[i]);
            // 更新
            if (i == currentEnd) {
                times++;
                currentEnd = maxReach;
                if (currentEnd >= len - 1)
                    break;
            }
        }
        return times;
    }
};
```
## 总结

### 1. 买卖股票的最佳时机（LeetCode 121）
**问题描述**：找到一次交易（买入和随后的卖出）能获得的最大利润。

**解题思路**：
- **维护两个变量**：`minPrice`（迄今为止遇到的最低价格）和`maxProfit`（迄今为止可以获得的最大利润）。
- **遍历数组**：对于数组中的每个价格，更新`minPrice`。计算如果在该价格卖出的利润，并更新`maxProfit`。

**代码实现**：在单次遍历中同时更新这两个变量，保证了时间复杂度为O(n)，空间复杂度为O(1)。

### 2. 跳跃游戏（LeetCode 55）
**问题描述**：判断你是否能够到达数组的最后一个位置。

**解题思路**：
- **维护一个变量**`maxReach`表示从当前或之前的索引最远能到达的位置。
- **遍历数组**：如果当前索引超过了`maxReach`，说明存在一点使得无法前进到数组的更远位置，返回false。
- **及时更新**：更新`maxReach`为当前索引加上在该索引处能跳的最远距离。

**代码实现**：时间复杂度为O(n)，空间复杂度为O(1)。

### 3. 跳跃游戏 II（LeetCode 45）
**问题描述**：找出到达数组最后一个位置的最小跳跃次数。

**解题思路**：
- **类似宽度优先搜索**：维护当前跳跃可达的最远边界`currentEnd`和下一跳可能达到的最远距离`maxReach`。
- **遍历数组**：每次到达`currentEnd`时，意味着完成了一次跳跃，更新跳跃次数，并将`currentEnd`更新为`maxReach`。
- **预先判断**：如果在某次跳跃后`maxReach`已经大于或等于数组最后一个位置，则结束遍历，返回当前跳跃次数。

**代码实现**：保持O(n)的时间复杂度和O(1)的空间复杂度，每次更新都尽可能延伸到最远距离，同时记录跳跃次数。

