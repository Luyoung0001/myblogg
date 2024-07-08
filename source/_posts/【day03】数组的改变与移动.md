---
title: 【day03】数组的改变与移动
date: 2023-03-10 23:42:56
tags:
    - leetcode
    - 算法
    - 数据结构
categories: LeetCode
---

<!--more-->

## 题目 1：[最小操作次数使数组元素相等](https://leetcode.cn/problems/minimum-moves-to-equal-array-elements/)

> 给你一个长度为 n 的整数数组，每次操作将会使 n - 1 个元素增加 1 。返回让数组所有元素相等的最小操作次数。

示例 1：

> 输入：nums = [1,2,3]
输出：3
解释：
只需要3次操作（注意每次操作会增加两个元素的值）：
[1,2,3]  =>  [2,3,3]  =>  [3,4,3]  =>  [4,4,4]

思路：答案很简单，但是很有意思。简单来说，`n-1`个数同时加1，相当于另外一个数加1。想要的答案是，要求出操作的最少次数，使得最终所有的元素的值相同。如果做加法，会造成超时以及可能情况上的数据溢出。就比如下面的代码：

```java
class Solution {
    public int minMoves(int[] nums) {
        Arrays.sort(nums);
        int ans = 0;
        while(nums[0] != nums[nums.length-1]){
            ans++;
            addOne(nums);
            Arrays.sort(nums);
        }
        return ans;
    }
    public void addOne(int[] nums){
        for(int i = 0; i < nums.length-1; i++){
            nums[i]++;

        }
    }

}
```

假定有一个最小的数 `min`，我们给最大的数 `max` 减一，就会使得这些元素的整体差距减小。最终的情况是，所有的数都会减少到 `min`，这是就得到了相同的效果。那么操作了多少次呢？就是所有的数与 `min` 的差距的总和：

```java
class Solution {
    public int minMoves(int[] nums) {
        int min = Integer.MAX_VALUE;
        // 求出最小值
        for(int num : nums){
            min = Math.min(min,num);
        }
        // 求和
        int count = 0;
        for(int num: nums){
            count += num-min;
        }
        return count;
    }
}
```

## 题目 2：[非递减数列](https://leetcode.cn/problems/non-decreasing-array/)

> 给你一个长度为 n 的整数数组 nums ，请你判断在 最多 改变 1 个元素的情况下，该数组能否变成一个非递减数列。
我们是这样定义一个非递减数列的： 对于数组中任意的 i (0 <= i <= n-2)，总满足 nums[i] <= nums[i + 1]。

示例 1：

> 输入: nums = [4,2,3]
输出: true
解释: 你可以通过把第一个 4 变成 1 来使得它成为一个非递减数列。

```java
class Solution {
    public boolean checkPossibility(int[] nums) {
        int count = 0;
        for (int i = 1; i < nums.length; i++) {
            if (nums[i] < nums[i - 1]) {
                if (i == 1 || nums[i] >= nums[i - 2]) {
                    nums[i - 1] = nums[i];
                } else {
                    nums[i] = nums[i - 1];
                }
                if (++count > 1) {
                    return false;
                }
            }
        }
        return true;
    }
}
```

### 题目 3：[移动零](https://leetcode.cn/problems/move-zeroes/)

> 给定一个数组 nums，编写一个函数将所有 0 移动到数组的末尾，同时保持非零元素的相对顺序。
请注意 ，必须在不复制数组的情况下原地对数组进行操作。

示例 1：

> 输入: nums = [0,1,0,3,12]
输出: [1,3,12,0,0]

思路很简单，遍历一遍：

```java
class Solution {
    public void moveZeroes(int[] nums) {
        int j=0;
        for(int i = 0; i < nums.length; i++){
            if(nums[i] != 0){
                int temp = nums[i];
                nums[i] = nums[j];
                nums[j++] = temp;
            }
        }
    }
}
```


