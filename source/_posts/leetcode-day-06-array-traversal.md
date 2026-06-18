---
title: "LeetCode 专题训练 Day 06：数组遍历"
date: 2023-03-13 16:54:42
tags:
  - "LeetCode"
  - "array"
categories:
  - "Algorithms"
---

<!--more-->

## 题目 1：[最大连续 1 的个数](https://leetcode.cn/problems/max-consecutive-ones/)

> 给定一个二进制数组 nums ， 计算其中最大连续 1 的个数。

示例 1：

>  输入：nums = [1,1,0,1,1,1]
输出：3
解释：开头的两位和最后的三位都是连续 1 ，所以最大连续 1 的个数是 3.

思路 1：很简单，直接模拟：

```java
class Solution {
    public int findMaxConsecutiveOnes(int[] nums) {
//        输入：nums = [1,0,1,1,0,1]
//        输出：2
        int len = nums.length;
        int max = 0;
        int end = 0;
        for(int i = 0; i < len; i++){
            if(nums[i] == 1){
                end = i;
                while(nums[end] == 1){
                    end++;
                }
                max = Math.max(max,end-i);
            }
            i = end;
        }
        return max;
    }
}
```
## 题目 2：[提莫攻击](https://leetcode.cn/problems/teemo-attacking/)

> 在《英雄联盟》的世界中，有一个叫 “提莫” 的英雄。他的攻击可以让敌方英雄艾希（编者注：寒冰射手）进入中毒状态。
当提莫攻击艾希，艾希的中毒状态正好持续 duration 秒。
正式地讲，提莫在 t 发起攻击意味着艾希在时间区间 [t, t + duration - 1]（含 t 和 t + duration - 1）处于中毒状态。如果提莫在中毒影响结束 前 再次攻击，中毒状态计时器将会 重置 ，在新的攻击之后，中毒影响将会在 duration 秒后结束。
给你一个 非递减 的整数数组 timeSeries ，其中 timeSeries[i] 表示提莫在 timeSeries[i] 秒时对艾希发起攻击，以及一个表示中毒持续时间的整数 duration 。
返回艾希处于中毒状态的 总 秒数。

示例 1：

> 输入：timeSeries = [1,4], duration = 2
输出：4
解释：提莫攻击对艾希的影响如下：
>- 第 1 秒，提莫攻击艾希并使其立即中毒。中毒状态会维持 2 秒，即第 1 秒和第 2 秒。
>- 第 4 秒，提莫再次攻击艾希，艾希中毒状态又持续 2 秒，即第 4 秒和第 5 秒。
艾希在第 1、2、4、5 秒处于中毒状态，所以总中毒秒数是 4 。


[思路 1：](https://leetcode.cn/problems/teemo-attacking/solution/ti-mo-gong-ji-by-leetcode-solution-6p4k/)单次扫描：



```java

class Solution {
    public int findPoisonedDuration(int[] timeSeries, int duration) {
        int ans = 0;
        int expired = 0;
        for (int i = 0; i < timeSeries.length; ++i) {
            if (timeSeries[i] >= expired) {
                ans += duration;
            } else {
                ans += timeSeries[i] + duration - expired;
            }
            expired = timeSeries[i] + duration;
        }
        return ans;
    }
}
```





## 题目 3：[第三大的数](https://leetcode.cn/problems/third-maximum-number/)

> 给你一个非空数组，返回此数组中 第三大的数 。如果不存在，则返回数组中最大的数。

示例 1：

> 输入：[3, 2, 1]
输出：1
解释：第三大的数是 1 。

思路 1： 直接模拟：

```java
class Solution {
    public int thirdMax(int[] nums) {
        int len = nums.length;
        Arrays.sort(nums);

        int tag = 1;
        for(int i = len-1; i >= 0; i--){
            if(i-1 >= 0 && nums[i] == nums[i-1]){
                continue;
            }else{
                tag++;
            }
            if(i-1 >= 0 && tag == 3){
                return nums[i-1];
            }
        }
        return nums[len-1];
    }
}
```

## 题目 4： [三个数的最大乘积](https://leetcode.cn/problems/maximum-product-of-three-numbers/)

> 给你一个整型数组 nums ，在数组中找出由三个数组成的最大乘积，并输出这个乘积。

示例 1：
> 输入：nums = [1,2,3,4]
输出：24

思路 1：先排序，然后直接返回：

```java
class Solution {
    public int maximumProduct(int[] nums) {
       int len = nums.length;
        Arrays.sort(nums);
        return Math.max(nums[0]*nums[1]*nums[len-1],nums[len-1]*nums[len-2]*nums[len-3]);
    }
}
```








