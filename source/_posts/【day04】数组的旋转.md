---
title: 【day04】数组的旋转
date: 2023-03-11 14:48:46
tags:
    - leetcode
    - 算法
    - 数据结构
categories: LeetCode
---

<!--more-->

## 题目 1：[旋转数组](https://leetcode.cn/problems/rotate-array/)

> 定一个整数数组 nums，将数组中的元素向右轮转 k 个位置，其中 k 是非负数。

示例 1：

> 输入: nums = [1,2,3,4,5,6,7], k = 3
输出: [5,6,7,1,2,3,4]
解释:
向右轮转 1 步: [7,1,2,3,4,5,6]
向右轮转 2 步: [6,7,1,2,3,4,5]
向右轮转 3 步: [5,6,7,1,2,3,4]

思路 1：创造一个新的数组，将转化后的数组再重新赋值给 nums：

```java
class Solution {
    public void rotate(int[] nums, int k) {
        int[] ans = new int[nums.length];
        for(int i = 0; i< nums.length; i++){
            ans[(i+k)%nums.length] = nums[i];
        }
        for(int i = 0; i < nums.length; i++){
            nums[i] = ans[i];
        }
    }
}
```
但是这种算法的空间复杂度为 O(n)，因此可以考虑使用反转数组的方法原地工作：

```java
class Solution {
    public void rotate(int[] nums, int k) {
        k %= nums.length;
        reverse(0,nums.length-1,nums);
        reverse(0,k-1,nums);
        reverse(k,nums.length-1,nums);

    }
    public void reverse(int start, int end, int[] nums){
        for(int i = start; 2*i < end+start; i++){
            int temp = nums[i];
            nums[i]= nums[end+start-i];
            nums[end+start-i] = temp;
        }
    }
}
```

## 题目 2：[旋转函数](https://leetcode.cn/problems/rotate-function/)

> 给定一个长度为 n 的整数数组 nums 。
假设 arrk 是数组 nums 顺时针旋转 k 个位置后的数组，我们定义 nums 的 旋转函数  F 为：
F(k) = 0 * arrk[0] + 1 * arrk[1] + ... + (n - 1) * arrk[n - 1]
返回 F(0), F(1), ..., F(n-1)中的最大值 。
生成的测试用例让答案符合 32 位 整数。

示例 1：

> 输入: nums = [4,3,2,6]
输出: 26
解释:
F(0) = (0 * 4) + (1 * 3) + (2 * 2) + (3 * 6) = 0 + 3 + 4 + 18 = 25
F(1) = (0 * 6) + (1 * 4) + (2 * 3) + (3 * 2) = 0 + 4 + 6 + 6 = 16
F(2) = (0 * 2) + (1 * 6) + (2 * 4) + (3 * 3) = 0 + 6 + 8 + 9 = 23
F(3) = (0 * 3) + (1 * 2) + (2 * 6) + (3 * 4) = 0 + 2 + 12 + 12 = 26
所以 F(0), F(1), F(2), F(3) 中的最大值是 F(3) = 26 。

思路：和上一道题一样，旋转完后，我们找最大的函数值就行：

```java
class Solution {
    public int maxRotateFunction(int[] nums) {

        int ans = 0;
        for(int j = 0; j < nums.length; j++){
            ans += j*nums[j];
        }
        int max = ans;
        for(int i = 0; i < nums.length-1; i++){
            ans = 0;
            rotate(nums,1);
            for(int j = 0; j < nums.length; j++){
                ans += j*nums[j];
            }
            max = Math.max(ans,max);
        }
        return max;
    }
    public void rotate(int[] nums, int k) {
        reverse(0,nums.length-1,nums);
        reverse(0,k-1,nums);
        reverse(k,nums.length-1,nums);

    }
    public void reverse(int start, int end, int[] nums){
        for(int i = start; 2*i < end+start; i++){
            int temp = nums[i];
            nums[i]= nums[end+start-i];
            nums[end+start-i] = temp;
        }
    }
}
```
这种解法的时间复杂度为 O(n^2)，因此最后一个测试会超时：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/6dfd7988b978472fae049d0128b11718_1720253841019.png?token=ANB4BCMVCU2WIVUW3T45ARLGRD644)
因此，得对算法进行改进。

观察，找规律：

>  nums: [A0,A1,A2,A3]


>  F0 = 0*A0 + 1*A1 + 2*A2 + 3*A3

>F1 = 0*A3 + 1*A0 + 2*A1 + 3*A2
    = F0 + A0 + A1 + A2 - 3*A3
    = F0 + sum-A3 - 3*A3
    = F0 + sum - 4*A3
>

>  F2 = 0*A2 + 1*A3 + 2*A0 + 3*A1
>     = F1 + A3 + A0 + A1 - 3*A2
>     = F1 + sum - 4*A2

>    F3 = 0*A1 + 1*A2 + 2*A3 + 3*A0
    = F2 + A2 + A3 + A0 - 3*A1
    = F2 + sum - 4*A1

抽象化:

    F(i) = F(i-1) + sum - n * A(n-i)


```java
class Solution {
    public int maxRotateFunction(int[] nums) {
        int sum = 0, f = 0, n = nums.length, ans = 0;
        for (int i = 0; i < n; i++) {
            sum += nums[i];
            f += i * nums[i];
        }
        ans = f;
        for (int i = 1; i < n; i++) {
            f = f + sum - n * nums[n - i];
            ans = Math.max(ans, f);
        }
        return ans;
    }
}
```







