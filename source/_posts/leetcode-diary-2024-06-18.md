---
title: "LeetCode 刷题日志：2024-06-18"
date: 2024-06-18 17:42:47
tags:
  - "LeetCode"
  - "algorithms"
  - "data structures"
categories:
  - "Algorithms"
---

<!--more-->

## 76. 最小覆盖子串
这道题目的思路是利用哈希表来存储字符串的字符出现的频率，然后利用滑动窗口来确定是否包含目标串，如果没有包含，就继续滑动右指针。当子串已经包含的时候，就收缩窗口，更新答案，以及右移 left。

```cpp
class Solution {
public:
    string minWindow(string s, string t) {
        // 滑动窗口
        int left = 0, right = 0;
        unordered_map<char, int> s_map, t_map;
        char temp, c;
        for (int ch : t) {
            t_map[ch]++;
        }
        int reque = t_map.size();
        int formed = 0;

        vector<int> ans = {0, 0, -1}; // left,right,len

        while (right < s.size()) {
            temp = s[right];
            s_map[temp]++;
            // 拓展右指针
            if (t_map.find(temp) != t_map.end() && s_map[temp] == t_map[temp]) {
                formed++;
            }
            // 左指针收缩
            while (left <= right && formed == reque) {
                c = s[left];
                // 更新答案
                if (ans[2] == -1 || ans[2] > right - left + 1) {
                    ans[0] = left;
                    ans[1] = right;
                    ans[2] = right - left + 1;
                }
                // 缩小
                s_map[c]--;
                if (t_map.find(c) != t_map.end() && s_map[c] < t_map[c]) {
                    formed--;
                }
                left++;
            }
            right++;
        }
        return ans[2] == -1 ? "" : s.substr(ans[0], ans[2]);
    }
};
```
## 53. 最大子数组和
动态规划，确定状态：`current_sum` 为 当前 `num[i]` 为结尾最大和子串；状态转移方程：`current_sum = max(nums[i], current_sum + nums[i])`，因此代码不难写出：

```cpp
class Solution {
public:
    int maxSubArray(vector<int>& nums) {
        int current_sum = nums[0]; // 初始化为第一个元素
        int max_sum = nums[0];     // 初始化为第一个元素

        for (int i = 1; i < nums.size(); i++) {
            current_sum = max(nums[i], current_sum + nums[i]);
            max_sum = max(max_sum, current_sum);
        }

        return max_sum;
    }
};

```
## 56. 合并区间
思路是首先排序，然后进行合并。合并时满足：merged.back()[1] >= interval[0]；否则，直接存储：

```cpp
class Solution {
public:
    vector<vector<int>> merge(vector<vector<int>>& intervals) {
        if (intervals.empty())
            return {};

        // 对区间进行排序，根据每个区间的起始位置
        sort(intervals.begin(), intervals.end(),
             [](const vector<int>& a, const vector<int>& b) {
                 return a[0] < b[0];
             });

        vector<vector<int>> merged;
        for (auto& interval : intervals) {
            // 如果merged为空或者当前区间不与merged中的最后一个区间重叠
            if (merged.empty() || merged.back()[1] < interval[0]) {
                merged.push_back(interval);
            } else {
                // 否则，有重叠，进行合并
                merged.back()[1] = max(merged.back()[1], interval[1]);
            }
        }
        return merged;
    }
};
```
## 189. 轮转数组
三次翻转就可以：

```cpp
class Solution {
public:
    void rotate(vector<int>& nums, int k) {
        int n = nums.size();
        k = k % n; // 处理 k 大于数组长度的情况
        reverse(nums.begin(), nums.end()); // 反转整个数组
        reverse(nums.begin(), nums.begin() + k); // 反转前 k 个元素
        reverse(nums.begin() + k, nums.end()); // 反转剩下的元素
    }
};
```

## 238. 除自身以外数组的乘积

这里使用到了前缀积，然后前缀积乘以后缀积，就得到了当前下标的解：

```cpp
class Solution {
public:
    vector<int> productExceptSelf(vector<int>& nums) {
        int n = nums.size();
        vector<int> ans(n, 1);
        // 前缀积
        int preMul = 1;
        for (int i = 0; i < n; i++) {
            ans[i] = preMul;
            preMul *= nums[i];
        }

        // 后缀积
        int postMul = 1;
        for (int i = n - 1; i >= 0; i--) {
            ans[i] *= postMul;
            postMul *= nums[i];
        }

        return ans;
    }
};
```

## 总结（gpt4）

### 1. 最小覆盖子串（滑动窗口和哈希表）
这个问题通过滑动窗口和哈希表来解决。利用两个哈希表分别记录目标字符串 `t` 和当前窗口中的字符出现频率。通过扩展右边界来包含更多字符，一旦包含了所有目标字符，就尝试从左边界收缩窗口以寻找最小覆盖子串。这种方法有效地平衡了窗口的大小和包含目标字符串所需的所有字符。

### 2. 最大子数组和（动态规划）
这个问题使用了动态规划来解决。状态定义非常直观：`current_sum` 为以当前元素结尾的最大子数组和。通过状态转移方程 `current_sum = max(nums[i], current_sum + nums[i])` 来更新当前状态，同时维护一个全局的最大和 `max_sum`。

### 3. 合并区间（排序和贪心算法）
首先对区间按起始位置进行排序，然后依次遍历排序后的区间，根据当前区间的起始位置与已合并区间的结束位置的关系来决定是否需要合并。这种方法依赖于贪心策略，通过局部最优选择（合并重叠区间）来达到全局最优（合并所有重古区间）。

### 4. 轮转数组（数组翻转）
这个解法巧妙地使用了三次翻转来实现数组的旋转。首先翻转整个数组，然后翻转前 `k` 个元素，最后翻转剩余的元素。这种方法在技术上是直接且高效的，避免了复杂的数组操作。

### 5. 除自身以外数组的乘积（前缀和后缀积）
通过两次遍历数组，先计算前缀积，再计算后缀积，并将二者相乘得到最终结果。这种方法避免了使用除法，同时解决了包含零的情况。通过前缀和后缀积的结合，能够高效地计算出每个位置上的目标值。


