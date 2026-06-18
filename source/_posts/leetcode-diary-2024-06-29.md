---
title: "LeetCode 刷题日志：2024-06-29"
date: 2024-06-29 12:53:39
tags:
  - "LeetCode"
  - "algorithms"
  - "data structures"
categories:
  - "Algorithms"
---

<!--more-->

## 739. 每日温度
给定一个整数数组 temperatures ，表示每天的温度，返回一个数组 answer ，其中 answer[i] 是指对于第 i 天，下一个更高温度出现在几天后。如果气温在这之后都不会升高，请在该位置用 0 来代替。

示例 1:

> 输入: temperatures = [73,74,75,71,69,72,76,73]
> 输出: [1,1,4,2,1,1,0,0]

示例 2:

> 输入: temperatures = [30,40,50,60]
> 输出: [1,1,1,0]

示例 3:

> 输入: temperatures = [30,60,90]
> 输出: [1,1,0]

这道题目可以用单调栈，栈中递减保存着数组下标，当当前值大于栈顶的时候，存进 ans，然后 pop；一直重复这个过程。然后 push 下标：

```cpp
class Solution {
public:
    vector<int> dailyTemperatures(vector<int>& temperatures) {
        // 单调栈，；类似于消消乐
        int n = temperatures.size();
        stack<int> st;
        vector<int> ans(n, 0);
        for (int i = 0; i < n; i++) {
            while (!st.empty() && temperatures[st.top()] < temperatures[i]) {
                int p = st.top();
                st.pop();
                ans[p] = i-p;
            }
            st.push(i);
        }
        return ans;
    }
};
```

## 215. 数组中的第K个最大元素

利用优先队列，当前队列存在着 k 个最大的元素，如果是小堆，堆顶就是第 K 大的元素：

```cpp
class Solution {
public:
    int findKthLargest(vector<int>& nums, int k) {
        priority_queue<int, vector<int>, greater<int>> miniHeap;
        for (int num : nums) {
            miniHeap.emplace(num);
            if (miniHeap.size() > k) {
                miniHeap.pop();
            }
        }
        return miniHeap.top();
    }
};
```
## 347. 前 K 个高频元素

这个稍微有点复杂，和上一道题目类似，首先统计到 map，然后构建优先队列，队列中存储 `pair<int,int>`,按照 **频率** 进行排序。

最后将队列中元素全部弹出到答案就可以了：

```cpp
class Solution {
public:
    vector<int> topKFrequent(vector<int>& nums, int k) {
        // 优于O(n log n)，可以分步处理
        unordered_map<int, int> map;
        for (int num : nums) {
            map[num] += 1;
        }
        // 优先队列
        priority_queue<pair<int, int>, vector<pair<int, int>>, greater<pair<int, int>>> pq;
        for (auto& item : map) {
            pq.push(pair(item.second,item.first)); // (freq,value)

            if (pq.size() > k)
                pq.pop();
        }
        // 最终处理
        vector<int> ans;
        while (!pq.empty()) {
            ans.push_back(pq.top().second);
            pq.pop();
        }
        return ans;
    }
};
```

## 总结

### 1. 每日温度 (LeetCode 739)
此问题使用了**单调栈**的方法。单调栈是解决此类“下一个更大元素”问题的一个常见技术。通过维护一个从栈底到栈顶递减的栈，可以有效地找到每个元素右侧第一个更大的元素。

#### 解法：
- 使用一个栈来保存温度的索引。
- 遍历温度数组，对于每个温度，如果当前温度大于栈顶索引对应的温度，则计算天数差并更新答案数组。
- 将当前索引推入栈中，等待后续的比较。
- 如果栈中剩余元素，对应的答案默认为0（已初始化）。

这种方法的时间复杂度是 O(n)，因为每个元素最多入栈和出栈一次。

### 2. 数组中的第K个最大元素 (LeetCode 215)
此问题利用**最小堆**来解决。通过维护一个大小为 k 的最小堆，堆顶元素始终是当前遇到的第 k 大的元素。

#### 解法：
- 使用一个最小堆来保存最大的 k 个元素。
- 遍历数组，将每个元素加入最小堆。
- 如果堆的大小超过 k，则弹出堆顶元素（最小元素）。
- 遍历完成后，堆顶元素即为第 k 大的元素。

这种方法的时间复杂度为 O(n log k)，其中 n 是数组的长度。

### 3. 前 K 个高频元素 (LeetCode 347)
此问题首先使用一个哈希表来统计每个元素的频率，然后利用一个**最小堆**来找到频率最高的 k 个元素。

#### 解法：
- 使用一个哈希表来统计每个元素的频率。
- 使用一个最小堆（存储频率和元素对）来维护最高频率的 k 个元素。
- 遍历哈希表，将频率和元素的对推入最小堆。
- 如果堆的大小超过 k，则弹出堆顶元素（频率最小的元素）。
- 遍历完成后，将堆中的元素收集到结果数组。

这种方法的时间复杂度为 O(n log k)，其中 n 是数组的长度。


