---
title: 【day01】之N数之和
date: 2023-03-07 21:57:40
tags:
    - leetcode
    - 算法
    - 数据结构
categories: LeetCode
---

<!--more-->

## 题目1：[两数之和](https://leetcode.cn/problems/two-sum/)

> 给定一个整数数组 nums 和一个整数目标值 target，请你在该数组中找出 和为目标值 target  的那 两个
> 整数，并返回它们的数组下标。
>
> 你可以假设每种输入只会对应一个答案。但是，数组中同一个元素在答案里不能重复出现。
>
> 你可以按任意顺序返回答案。
>
示例 1：
> 输入：nums = [2,7,11,15], target = 9
输出：[0,1]
解释：因为 nums[0] + nums[1] == 9 ，返回 [0, 1] 。

这个题很简单，直接使用map：
## 答案1

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        HashMap<Integer, Integer> map = new HashMap<>();
        for(int i = 0; i < nums.length; i++){
            if(map.containsKey(target-nums[i])){
                return new int[]{
                    map.get(target-nums[i]),i
                    };
            }
            map.put(nums[i],i);
        }
        return new int[0];
    }
}
```
## 题目 2：[三数之和](https://leetcode.cn/problems/3sum/)

> 三数之和
给你一个整数数组 nums ，判断是否存在三元组 [nums[i], nums[j], nums[k]] 满足 i != j、i != k 且 j != k ，同时还满足 nums[i] + nums[j] + nums[k] == 0 。请你返回所有和为 0 且不重复的三元组。注意：答案中不可以包含重复的三元组。

示例 1：

> 输入：nums = [-1,0,1,2,-1,-4]
输出：[[-1,-1,2],[-1,0,1]]
解释：
nums[0] + nums[1] + nums[2] = (-1) + 0 + 1 = 0 。
nums[1] + nums[2] + nums[4] = 0 + 1 + (-1) = 0 。
nums[0] + nums[3] + nums[4] = (-1) + 2 + (-1) = 0 。
不同的三元组是 [-1,0,1] 和 [-1,-1,2] 。
注意，输出的顺序和三元组的顺序并不重要。

这个题目就是特殊的两数之和，因此可以有下面的思路：

### 答案 1：

```java
class Solution {
    public List<List<Integer>> threeSum(int[] nums) {
        List<List<Integer>> ans = new ArrayList<>();
        for (int i = 0; i < nums.length; i++) {
            // 嵌套的两数之和

            Map<Integer, Integer> map = new HashMap<>();
            for (int j = 1; (j < nums.length) && (j != i); j++) {
                List<Integer> etem = new ArrayList<>();
                int temp = -nums[i] - nums[j];
                if (map.containsKey(temp)) {
                    etem.add(nums[i]);
                    etem.add(nums[j]);
                    etem.add(temp);
                    // 如果不在 ans里面,就放进去
                    if (!isIn(etem, ans)) {
                        ans.add(etem);
                    }
                }
                map.put(nums[j], j); // map查询比 list快很多
            }
        }
        return ans;
    }

    public boolean isIn(List<Integer> list, List<List<Integer>> lists) {
        // 判断list 是否在lists中
        for (List<Integer> lis : lists) {
            if (list_equals(list, lis)) {
                return true;
            }
        }
        return false;
    }

    public boolean list_equals(List<Integer> l1, List<Integer> l2) {
        Integer[] nums1 = l1.toArray(new Integer[0]);
        Integer[] nums2 = l2.toArray(new Integer[0]);
        if (nums1.length != nums2.length) {
            return false;
        }
        // 排序
        Arrays.sort(nums1);
        Arrays.sort(nums2);
        for (int i = 0; i < nums1.length; i++) {
            if (nums1[i] != nums2[i]) {
                return false;
            }
        }
        return true;
    }
}
```
很显然这个答案的时间复杂度过于高，为 O(n^4)，会报超时：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/78400dd501f44d24987bfeaad16ef0a5_1720253841019.png?token=ANB4BCNVSHDTPIGX5ESBUQTGRD65G)
因此可以改进思路，先对数组排序，然后设置双指针进行移动。在得到一个答案之后，必须得保证左端元素的值和原来不同：
### 答案 2：

```java
class Solution {
    public List<List<Integer>> threeSum(int[] nums) {
        List<List<Integer>> result = new LinkedList<>();
        Arrays.sort(nums);
        // 对变量进行排序，如果第一个元素大于 0，那么 result 肯定是空的
        if (nums[0] > 0) return result;
        int i = 0;
        while (i < nums.length - 2) {
            TwoSum(i, nums, result);
            int temp = nums[i];
            while (i < nums.length - 2 && nums[i] == temp) {
                i++;
            }
        }
        return result;
    }
    public void TwoSum(int i, int[] nums, List<List<Integer>> result) {
        int p1 = i + 1;
        int p2 = nums.length - 1;
        // 双指针遍历所有可能情况
        while (p1 < p2) {
            if (nums[p1] + nums[p2] + nums[i] > 0) {
                p2--;
            } else if (nums[p1] + nums[p2] + nums[i] < 0) {
                p1++;
            } else {
                LinkedList<Integer> list = new LinkedList<Integer>();
                list.add(nums[p1]);
                list.add(nums[p2]);
                list.add(nums[i]);
                result.add(list);

                int temp1 = nums[p1];
                // 移动左端指针
                while (p1 < p2 && nums[p1] == temp1) {
                    p1++;
                }
            }
        }
    }
}
```
时间复杂度为O(N^2)。继续对上述答案简化，就可以完全得到一个双指针的答案：
### 答案 3：

```java
class Solution {
    public List<List<Integer>> threeSum(int[] nums) {   //输入：nums = [-1,0,1,2,-1,-4]--->[-1,-1,0,1,2,4]
                                                        //输出：[[-1,-1,2],[-1,0,1]]
        Arrays.sort(nums);
        List<List<Integer>> ans = new ArrayList<>();
        if(nums[0] > 0) return ans;
        int L = 0;
        int R = 0;
        for(int i = 0; i < nums.length; i++){
            if(nums[i] > 0) return ans;
            if(i >= 1 && nums[i] == nums[i-1]) continue;
            L = i+1;
            R = nums.length-1;
            while(L < R){
                int temp = nums[i]+nums[L]+nums[R];
                if( temp == 0){
                    List<Integer> list = new ArrayList();
                    list.add(nums[i]);
                    list.add(nums[L]);
                    list.add(nums[R]);
                    ans.add(list);
                    while(L < R && nums[L] == nums[L+1]) L++;
                    while(L < R && nums[R] == nums[R-1]) R--;
                    L++;
                    R--;
                }else if(temp > 0){
                    R--;
                }else{
                    L++;
                }
            }
        }
        return ans;
    }
}
```
时间复杂度为O(N^2)。

## 题目三：[四数之和](https://leetcode.cn/problems/4sum/)

> 18. 四数之和
给你一个由 n 个整数组成的数组 nums ，和一个目标值 target 。请你找出并返回满足下述全部条件且不重复的四元组 [nums[a], nums[b], nums[c], nums[d]] （若两个四元组元素一一对应，则认为两个四元组重复）：
0 <= a, b, c, d < n
a、b、c 和 d 互不相同
nums[a] + nums[b] + nums[c] + nums[d] == target
你可以按 任意顺序 返回答案 。

示例 1：

> 输入：nums = [1,0,-1,0,-2,2], target = 0
输出：[[-2,-1,1,2],[-2,0,0,2],[-1,0,0,1]]


仍然是三数之和的变种，但是情况就比较抽象：

```java
class Solution {
    public List<List<Integer>> fourSum(int[] nums, int target) {
        List<List<Integer>> lists = new ArrayList<List<Integer>>();
        if (nums == null || nums.length < 4) {
            return lists;
        }
        Arrays.sort(nums);
        int length = nums.length;
        for (int i = 0; i < length - 3; i++) {
            if (i > 0 && nums[i] == nums[i - 1]) continue;
            if ((long) nums[i] + nums[i + 1] + nums[i + 2] + nums[i + 3] > target) break;
            if ((long) nums[i] + nums[length - 3] + nums[length - 2] + nums[length - 1] < target) continue;

            for (int j = i + 1; j < length - 2; j++) {
                if (j > i + 1 && nums[j] == nums[j - 1]) continue;

                if ((long) nums[i] + nums[j] + nums[j + 1] + nums[j + 2] > target) break;

                if ((long) nums[i] + nums[j] + nums[length - 2] + nums[length - 1] < target) continue;
                int left = j + 1, right = length - 1;
                while (left < right) {
                    long sum = (long) nums[i] + nums[j] + nums[left] + nums[right];
                    if (sum == target) {
                        lists.add(Arrays.asList(nums[i], nums[j], nums[left], nums[right]));
                        while (left < right && nums[left] == nums[left + 1]) {
                            left++;
                        }
                        left++;
                        while (left < right && nums[right] == nums[right - 1]) {
                            right--;
                        }
                        right--;
                    } else if (sum < target) {
                        left++;
                    } else {
                        right--;
                    }
                }
            }
        }
        return lists;
    }
}
```
时间复杂度为 O(N^3)。



