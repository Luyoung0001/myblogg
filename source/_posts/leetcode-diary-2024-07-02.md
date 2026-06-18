---
title: "LeetCode 刷题日志：2024-07-02"
date: 2024-07-02 23:14:13
tags:
  - "LeetCode"
  - "algorithms"
  - "data structures"
categories:
  - "Algorithms"
---

<!--more-->

## 70. 爬楼梯
示例 1：

> 输入：n = 2
> 输出：2
> 解释：有两种方法可以爬到楼顶。
> 1. 1 阶 + 1 阶
> 2. 2 阶

示例 2：

> 输入：n = 3
> 输出：3
> 解释：有三种方法可以爬到楼顶。
> 1. 1 阶 + 1 阶 + 1 阶
> 2. 1 阶 + 2 阶
> 3. 2 阶 + 1 阶

状态定义：`dp[i]`，代表着当前阶梯到达的不同方法数。
状态转移方程：`dp[i] = dp[i-1]+dp[i-2]`，这意味着当前阶梯到达的不同方法数等于上一阶梯数+上上一级阶梯数的和。
初始化：`dp[0] = 1;dp[1] = 1`。经过空间复杂度优化后的代码如下：

```cpp
class Solution {
public:
    int climbStairs(int n) {
        int p = 1, q = 1, temp;
        for (int i = 2; i <= n; i++) {
            temp = q;
            q = q + p;
            p = temp;
        }
        return q;
    }
};
```

## 118. 杨辉三角

这道题目感觉和动态规划关系不大，直接用数学模拟：

```cpp
class Solution {
public:
    vector<vector<int>> generate(int numRows) {
        vector<vector<int>> ans;
        ans.emplace_back(vector<int>(1, 1));
        for (int i = 1; i < numRows; i++) {
            auto& prev = ans[i - 1];
            vector<int> temp;
            for (int j = 0; j < i + 1; j++) {
                if (j == 0 || j == i)
                    temp.push_back(1);
                else
                    temp.push_back(prev[j] + prev[j - 1]);
            }
            ans.push_back(std::move(temp));
        }
        return ans;
    }
};
```

## 198. 打家劫舍
这道题目的关键是找状态转移方程，状态定义：`dp[i]` 为打劫到 `i` 号房子的最大收益。状态转移方程为：`dp[i] = max(dp[i-2]+nums[i], dp[i]);`。状态初始化为：`ans[0] = nums[0];  ans[1] = max(nums[0], nums[1]);`：

```cpp
class Solution {
public:
    int rob(vector<int>& nums) {
        if (nums.size() == 1)
            return nums[0];
        vector<int> ans(nums.size(), 0);
        ans[0] = nums[0];
        ans[1] = max(nums[0], nums[1]);
        for (int i = 2; i < nums.size(); i++) {
            ans[i] = max(ans[i - 1], ans[i - 2] + nums[i]);
        }
        return max(ans[nums.size() - 1], ans[nums.size() - 2]);
    }
};
```

## 279. 完全平方数
这道题目的状态定义dp[i]为数字 i 的最小平方数个数，状态装方程为：`ans[i] = min(ans[i], ans[i - j * j] + 1);`。

比如：`[1,2,3,4,5,6,7,8,9,10,11...]`，对于 9，`dp[9]`可能为 `dp[9-3*3]+1 = 1;`。`dp[10]` 可能为 `dp[10-1*1]+1、dp[10-2*2]+1、dp[10-3*3]+1`中的最小值。

**这里面隐含了这样一个事实：dp[n]  = dp[n-i*i]+1，也就是说 n 的平方最小数为m(m < n)的平方数+1。**

```cpp
class Solution {
public:
    int numSquares(int n) {
        vector<int> ans(n + 1, INT_MAX);
        ans[0] = 0;
        for (int i = 1; i <= n; i++) {
            for (int j = 1; j * j <= i; j++) {
                // [1,2,3,4,5,6,7,8,9]
                ans[i] = min(ans[i], ans[i - j * j] + 1);
            }
        }
        return ans[n];
    }
};
```

## 总结

这些题目都涉及到动态规划的核心思想，即定义状态和状态转移方程来求解问题。下面是对每个问题的核心思路总结：

### 1. 爬楼梯（70）
- **核心思想**：爬楼梯问题实际上是斐波那契数列的应用。每一级台阶的爬法数是前两级台阶爬法数的和。
- **状态定义**：`dp[i]` 表示达到第 `i` 个台阶有多少种方法。
- **状态转移方程**：`dp[i] = dp[i-1] + dp[i-2]`
- **空间优化**：由于每一步只依赖前两个状态，可以使用两个变量滚动更新，降低空间复杂度至 O(1)。

### 2. 杨辉三角（118）
- **核心思想**：直接根据杨辉三角的定义，使用上一行来生成当前行。
- **实现方式**：每一行的首尾元素为1，中间的元素等于上一行的相邻两个元素之和。

### 3. 打家劫舍（198）
- **核心思想**：不能连续抢劫两个相邻的房子，因此每个房子的决策是基于前一个和前两个房子的最优决策。
- **状态定义**：`dp[i]` 表示到第 `i` 个房子时能抢劫到的最大金额。
- **状态转移方程**：`dp[i] = max(dp[i-1], dp[i-2] + nums[i])`
- **初始化**：`dp[0] = nums[0], dp[1] = max(nums[0], nums[1])`

### 4. 完全平方数（279）
- **核心思想**：给定一个数，求最少的完全平方数之和来表示该数。这个问题可以看作是一种组合问题，选择的数必须是完全平方数。
- **状态定义**：`dp[i]` 表示组成整数 `i` 需要的最少完全平方数的数量。
- **状态转移方程**：`dp[i] = min(dp[i], dp[i - j * j] + 1)`，其中 `j*j <= i`
- **初始化**：`dp[0] = 0`（0由0个平方数构成）

