---
title: 【day05】统计数组中的元素
date: 2023-03-12 15:37:15
tags: leetcode 算法 数据结构
categories: LeetCode
---

<!--more-->

## 题目1：[错误的集合](https://leetcode.cn/problems/set-mismatch/)

> 集合 s 包含从 1 到 n 的整数。不幸的是，因为数据错误，导致集合里面某一个数字复制了成了集合里面的另外一个数字的值，导致集合 丢失了一个数字 并且 有一个数字重复 。
给定一个数组 nums 代表了集合 S 发生错误后的结果。
请你找出重复出现的整数，再找到丢失的整数，将它们以数组的形式返回。

示例 1：

> 输入：nums = [1,2,2,4] 
> 输出：[2,3]

思路：因为需要计数，因此哈希表很合适

```java

class Solution {
    public int[] findErrorNums(int[] nums) {
        // 哈希
        Map<Integer, Integer> map = new HashMap<>();
        for(int i = 0; i < nums.length; i++){
            map.put(nums[i], map.getOrDefault(nums[i],0)+1);
        }
        int less = 0,more = 0;
        for(int i = 1; i <= nums.length; i++){
            if(map.getOrDefault(i,0) == 0){
                less = i;
            }
            if(map.getOrDefault(i,0) == 2){
                more = i;
            }
        }
        return new int[]{more, less};
    }
}
```
时间复杂度和空间复杂度都是 O(n)。

## 题目 2：[数组的度](https://leetcode.cn/problems/degree-of-an-array/)

> 给定一个非空且只包含非负数的整数数组 nums，数组的 度 的定义是指数组里任一元素出现频数的最大值。
你的任务是在 nums 中找到与 nums 拥有相同大小的度的最短连续子数组，返回其长度。


示例 1：

> 输入：nums = [1,2,2,3,1]
输出：2
解释：
输入数组的度是 2 ，因为元素 1 和 2 的出现频数最大，均为 2 。
连续子数组里面拥有相同度的有如下所示：
[1, 2, 2, 3, 1], [1, 2, 2, 3], [2, 2, 3, 1], [1, 2, 2], [2, 2, 3], [2, 2]
最短连续子数组 [2, 2] 的长度为 2 ，所以返回 2 。

思路：哈希表

```java
class Solution {
    public int findShortestSubArray(int[] nums) {
        Map<Integer, Integer> map = new HashMap<>();
        // 计数
        int max = 0;
        for (int num : nums) {
            map.put(num, map.getOrDefault(num, 0) + 1);
            max = Math.max(map.getOrDefault(num, 0), max);
        }
        // 最大的度即为 max
        int len = nums.length;
        int p = 0, tag = 0;
        for (int i = 0; i < nums.length; i++) {
            if (map.getOrDefault(nums[i], 0) == max) {
                p = i;
                tag = i;
                while (p < nums.length) {
                    if (nums[p] == nums[i]) {
                        tag = p;
                    }
                    p++;
                }
                len = Math.min(len, tag - i + 1);
                //查过之后，就破坏掉
                map.put(nums[i],0);
            }
        }
        return len;
    }
}
```
## 题目 3：[找到所有数组中消失的数字](https://leetcode.cn/problems/find-all-numbers-disappeared-in-an-array/)

> 给你一个含 n 个整数的数组 nums ，其中 nums[i] 在区间 [1, n] 内。请你找出所有在 [1, n] 范围内但没有出现在 nums 中的数字，并以数组的形式返回结果。
>
示例 1：
> 输入：nums = [4,3,2,7,8,2,3,1]
输出：[5,6]

思路1：简单题，哈希表

```java
class Solution {
    public List<Integer> findDisappearedNumbers(int[] nums) {
        // 模拟
        Map<Integer, Integer> map = new HashMap<>();
        List<Integer> ans = new ArrayList<>();
        for(int i = 0; i < nums.length; i++){
            map.put(nums[i], map.getOrDefault(nums[i],0)+1);
        }
        for(int i = 1;i <= nums.length; i++){
            if(map.getOrDefault(i,0) == 0){
                ans.add(i);
            }
        }
        return ans;
    }
}
```
[思路 2@这条街最靓的仔](https://leetcode.cn/problems/find-all-numbers-disappeared-in-an-array/comments/9743)：将所有正数作为数组下标，置对应数组值为负值。那么，仍为正数的位置即为（未出现过）消失的数字，简单题。

举个例子：

> 原始数组：[4,3,2,7,8,2,3,1]
> 
> 重置后为：[-4,-3,-2,-7,8,2,-3,-1]

结论：[8,2] 分别对应的index为[5,6]

```java
class Solution {
    public List<Integer> findDisappearedNumbers(int[] nums) {
        List<Integer> list = new ArrayList<>();
        for (int i = 0; i < nums.length; i++) {
            nums[Math.abs(nums[i]) - 1] = -Math.abs(nums[Math.abs(nums[i]) - 1]);
        }
        for (int i = 0; i < nums.length; i++) {
            if (nums[i] > 0) {
                list.add(i + 1);
            }
        }
        return list;
    }
}
```
## 题目 4：[数组中重复的数据](https://leetcode.cn/problems/find-all-duplicates-in-an-array/)

> 给你一个长度为 n 的整数数组 nums ，其中 nums 的所有整数都在范围 [1, n] 内，且每个整数出现 一次 或 两次
> 。请你找出所有出现 两次 的整数，并以数组形式返回。 你必须设计并实现一个时间复杂度为 O(n) 且仅使用常量额外空间的算法解决此问题。

示例 1：

> 输入：nums = [4,3,2,7,8,2,3,1]
输出：[2,3]

思路：简单题，和上一题的思路 2 基本一致，见注释：

```java
class Solution {
    public List<Integer> findDuplicates(int[] nums) {
        int len = nums.length;
        // [4,3,2,7,8,2,3,1]
        // 4 -3 -2 -7 -8 -3,-1,
        List<Integer> list = new ArrayList<>();
        for (int i = 0; i < len; i++) {
            int num = Math.abs(nums[i]);
            if(nums[num-1] > 0){
                nums[num-1] *= -1;
            }else{
                list.add(num);
            }
        }
        return list;
    }
}
```
## 题目 5：[缺失的第一个正数](https://leetcode.cn/problems/first-missing-positive/)

> 给你一个未排序的整数数组 nums ，请你找出其中没有出现的最小的正整数。
请你实现时间复杂度为 O(n) 并且只使用常数级别额外空间的解决方案。

示例 1：

> 输入：nums = [1,2,0]
输出：3

思路 1：题目是一道困难题目，先不管时间复杂度的话，就是模拟：

```java
class Solution {
    public int firstMissingPositive(int[] nums) {
     // 这个排序的时间复杂度为：
        Arrays.sort(nums);
        int tag = 1;
        for(int i = 0; i < nums.length; i++){
           if(nums[i] <= 0 || (i != 0)&&(nums[i] == nums[i-1]) ){
               continue;
           }
           if(nums[i] != tag){
               return tag;
           }else{
               tag++;
           }  
        }
        return tag;
    }
}
```
但是题目要求是时间复杂度为O(n)，思路1 的复杂度为O(nlgn)，不符合要求，得另辟新径：

```java
class Solution {
    public int firstMissingPositive(int[] nums) {
        int len = nums.length;
        // 找 1
        int tag = 0;
        for(int i = 0; i < len; i++){
            if(nums[i] == 1){
                tag = 1;
            }
        }
        if(tag == 0) return 1;
        // 打上标记1
        for(int i = 0; i < len; i++){
            if(nums[i] <= 0 || nums[i] > len){
                nums[i] = 1;
            }
        }
        // 2,3,4,1,2,4
        for(int i = 0; i < len; i++){
            int temp = Math.abs(nums[i])-1;
            nums[temp] = -Math.abs(nums[temp]);
        }
        // 遍历
        for(int i = 0; i < len; i++){
            if(nums[i] > 0){
                return i+1;
            }
        }
        return len+1;
    }
}
```
## 题目 6：[丢失的数字](https://leetcode.cn/problems/missing-number/)

> 给定一个包含 [0, n] 中 n 个数的数组 nums ，找出 [0, n] 这个范围内没有出现在数组中的那个数。

示例 1：

> 输入：nums = [3,0,1]
输出：2
解释：n = 3，因为有 3 个数字，所以所有的数字都在范围 [0,3] 内。2 是丢失的数字，因为它没有出现在 nums 中。


思路 1：可以使用位运算：

```java
class Solution {
    public int missingNumber(int[] nums) {
        int ans = nums.length;
        for(int i = 0; i < nums.length; i++){
            ans = ans^nums[i]^i;
        }
        return ans;
    }
}
```
思路 2：思路1 看起来并没有太大意义，我们应该追求思想的统一，因此继续使用上面几道题目中，哈希思想的解法：

```java
class Solution {
    public int missingNumber(int[] nums) {
        int n = nums.length;
        boolean zero = false;
        for (int num : nums) {
            int abs = Math.abs(num);
            if (abs < n) {
                if (nums[abs] == 0) zero = true;
                nums[abs] = -nums[abs];
            }
        }
        for (int i = 0; i < n; i++) {
            if (nums[i] > 0 || (nums[i] == 0 && !zero)) return i;
        }
        return n;
    }
}
```











