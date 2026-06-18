---
title: "LeetCode 刷题日志：2024-06-25"
date: 2024-06-25 22:54:05
tags:
  - "LeetCode"
  - "algorithms"
  - "data structures"
categories:
  - "Algorithms"
---

<!--more-->

## 704. 二分查找
这种题有多种写法，我认为应该固定一种写法，从而养成自己的编程直觉，不然很容易造成思维混乱。二分查找说难不难，说简单也不简单。**最难的点就是处理边界条件**，有时候多个等号会通过，少个等号就通过不了关键是还不知道为什么能过。

示例 1:

> 输入: nums = [-1,0,3,5,9,12], target = 9
> 输出: 4
> 解释: 9 出现在 nums 中并且下标为 4

示例 2:

> 输入: nums = [-1,0,3,5,9,12], target = 2
> 输出: -1
> 解释: 2 不存在 nums 中因此返回 -1

这道题目是最简单的二分查找，很经典。面对这样的题目，首先就得明白，**自己要在开区间搜索还是闭区间搜索，这对于边界条件的把握很关键**。

### 开区间
那就是在`[left, right)`中查找，此时，right 可以取到 nums 的边界之后，即 right 可以取 `nums.size()`。一旦决定要用开区间来解决问题，那么边界条件的确定就很关键，要不要加等号？考虑例子 `nums = [1], target = 2`。此时应该返回 -1。如果加了等号，即` while(left <= right)`，那么 下次还会继续再次执行 `while(){}, 而 mid = 1`,就会造成 **越界访问**：

```cpp
// nums = [1], target = 2
class Solution {
public:
    int search(vector<int>& nums, int target) {
        int left = 0, right = nums.size(), mid;
        while (left <= right) {
            mid = left + (right - left) / 2;
            if (nums[mid] == target)
                return mid;
            if (nums[mid] < target)
                left = mid + 1;
            else
                right = mid;
        }
        return -1;
    }
};
```
因此应该形成这种认知：**使用开区间搜索的时候，mid 可能会越界，为了防止越界，应该禁止下次访问**：即 `while(left < right)`：

```cpp
class Solution {
public:
    int search(vector<int>& nums, int target) {
        int left = 0, right = nums.size(), mid;
        while (left < right) {
            mid = left + (right - left) / 2;
            if (nums[mid] == target)
                return mid;
            if (nums[mid] < target)
                left = mid + 1;
            else
                right = mid;
        }
        return -1;
    }
};
```

### 闭区间
闭区间搜索是在`[left, right]`中搜索，mid 的取值是不会越界的，因此 `while(left <= right)`是没有任何问题的，思路是同时缩进 left、right，直到 left = right：

```cpp
class Solution {
public:
    int search(vector<int>& nums, int target) {
        int left = 0, right = nums.size()-1, mid;
        while (left <= right) {
            mid = left + (right - left) / 2;
            if (nums[mid] == target)
                return mid;
            if (nums[mid] < target)
                left = mid + 1;
            else
                right = mid-1;
        }
        return -1;
    }
};
```

因此我认为应该养成这两种思维中一种，看个人喜欢哪种逼近方式，但必须选一个。我个人喜欢闭区间，这种不会产生越界访问的风险，而且很直观。

## 35. 搜索插入位置
示例 1:

> 输入: nums = [1,3,5,6], target = 5
> 输出: 2

示例 2:

> 输入: nums = [1,3,5,6], target = 2
> 输出: 1

示例 3:

> 输入: nums = [1,3,5,6], target = 7
> 输出: 4

此时，就应该从上一道题目中发展出这道题目的思路了，我自己采用闭区间搜索。

首先这道题目基本功能是搜索一个目标值的索引，因此和上一道题目一致：

```cpp
class Solution {
public:
    int searchInsert(vector<int>& nums, int target) {
        int left = 0, right = nums.size() - 1, mid;
        while (left <= right) {
            mid = left + (right - left) / 2;
            if (nums[mid] == target)
                return mid;
            if (nums[mid] < target)
                left = mid + 1;
            else
                right = mid - 1;
        }
        return -1;
    }
};
```

但是，原题要求找不到的时候，要求返回应该插入的位置，因此在返回到 -1 的位置，返回left。为什么可以这样？

我们应该注意到，找不到的时候，left、right 是什么状态。假设 `nums = [1,3,5,6]，target = 4`：

```cpp
class Solution {
public:
    int searchInsert(vector<int>& nums, int target) {
        int left = 0, right = nums.size() - 1, mid;
        while (left <= right) {
            mid = left + (right - left) / 2;
            if (nums[mid] == target)
                return mid;
            if (nums[mid] < target)
                left = mid + 1;
            else
                right = mid - 1;
        }
        return -1;
    }
};
```

- left = 0，right = 3：mid  = 1，nums[mid] = 3 < 4，left = 2;
- left = 2，right = 3：mid = 2，nums[mid] = 5 > 4，right = 2；
- left = 2，right = 2：mid = 2，nums[mid] = 5 > 4，right = 1;

退出循环，可以看到，此时 left 的位置是大于 target 的第一个位置，这来源于 `if (nums[mid] < target)  left = mid + 1;`，因此，我们需要在最后 `return left`。

```cpp
class Solution {
public:
    int searchInsert(vector<int>& nums, int target) {
        int left = 0, right = nums.size() - 1, mid;
        while (left <= right) {
            mid = left + (right - left) / 2;
            if (nums[mid] == target)
                return mid;
            if (nums[mid] < target)
                left = mid + 1;
            else
                right = mid - 1;
        }
        return left;
    }
};
```

## 744. 寻找比目标字母大的最小字母
示例 1：

> 输入: letters = ["c", "f", "j"]，
> target = "a" 输出: "c"
> 解释：letters 中字典上比 'a' 大的最小字符是 'c'。

示例 2:

> 输入: letters = ["c","f","j"], target = "c"
> 输出: "f"
> 解释：letters 中字典顺序上大于 'c' 的最小字符是 'f'。

示例 3:

> 输入: letters = ["x","x","y","y"], target = "z"
> 输出: "x"
> 解释：letters 中没有一个字符在字典上大于 'z'，所以我们返回 letters[0]。

这道题目依然来源于第一道题，首先如果能搜索到，那么只需要返回这个字母的下一个不一样的字母就行：

```cpp
class Solution {
public:
    char nextGreatestLetter(vector<char>& letters, char target) {
        int left = 0, right = letters.size() - 1, mid;
        while (left <= right) {
            mid = left + (right - left) / 2;
            // 如果找到，那么就返回这个字母的下一个不一样的字母就行
            if (letters[mid] == target) {
                while (mid < letters.size() && letters[mid] == target) {
                    mid++;
                }
                // 如果越界，说明后面没有比它打的，返回第一个
                if (mid >= letters.size())
                    return letters[0];
                else
                    return letters[mid];
            }
            if (letters[mid] < target) {
                left = mid + 1;
            } else
                right = mid - 1;
        }
        return letters[0];
    }
};
```

当然上面代码是有问题的，但是它目前能解决基本问题，接下来继续改造。

前面说过，如果没搜索到，那么 left 此时指向的就是比目标大的第一个位置，因此可以返回 left。但是如果此时是这一张输入：`letters = ["x","x","y","y"]，target = "z"`，此时 left 就会越界，此时应该返回 letters[0]。因此：

```cpp
class Solution {
public:
    char nextGreatestLetter(vector<char>& letters, char target) {
        int left = 0, right = letters.size() - 1, mid;
        while (left <= right) {
            mid = left + (right - left) / 2;
            if (letters[mid] == target) {
                while (mid < letters.size() && letters[mid] == target) {
                    mid++;
                }
                if (mid >= letters.size())
                    return letters[0];
                else
                    return letters[mid];
            }
            if (letters[mid] < target) {
                left = mid + 1;
            } else
                right = mid - 1;
        }
        return letters[left%letters.size()];
    }
};
```

这样就把整个答案完善了。我认为解决这样的问题，必须记住甚至背过最基本的题型以及解法，然后遇到类似的题目可以从它身上发散，**做二分，就是在做边界条件**。


## 34. 在排序数组中查找元素的第一个和最后一个位置

示例 1：

> 输入：nums = [5,7,7,8,8,10], target = 8
> 输出：[3,4]

示例 2：

> 输入：nums = [5,7,7,8,8,10], target = 6
> 输出：[-1,-1]

示例 3：

> 输入：nums = [], target = 0
> 输出：[-1,-1]

这道题目虽然是中等难度的题目，但是还是很简单的，难点就是把握边界条件。首先查找：

```cpp
class Solution {
public:
    vector<int> searchRange(vector<int>& nums, int target) {
        int left = 0, right = nums.size() - 1, mid, pos = -1;
        while (left <= right) {
            mid = left + (right - left) / 2;
            if (nums[mid] == target) {
                pos = mid;
                break;
            }
            if (nums[mid] < target)
                left = mid + 1;
            else
                right = mid - 1;
        }
        // 接着处理

    }
};
```

如果没有找到，那么返回 `{-1, -1}`；如果找到了，那么就很好，我们只需要从这个值向两边查找就行：

```cpp
if (pos == -1)
            return {-1, -1};

        int p1 = pos, p2 = pos; // 搜索边界
        while (p1 >= 0) {
            if (nums[p1] == target) {
                p1--;
            } else
                break;
        }
        while (p2 < nums.size()) {
            if (nums[p2] == target) {
                p2++;
            } else
                break;
        }
        return {p1+1, p2-1};
```

因为从 `while` 中出来时，p1、p2的值一定是超越正确位置一个身位的，因此必须返回 `{p1+1, p2-1};`。因此整个题目的解法就是：

```cpp
class Solution {
public:
    vector<int> searchRange(vector<int>& nums, int target) {
        int left = 0, right = nums.size() - 1, mid, pos = -1;
        while (left <= right) {
            mid = left + (right - left) / 2;
            if (nums[mid] == target) {
                pos = mid;
                break;
            }
            if (nums[mid] < target)
                left = mid + 1;
            else
                right = mid - 1;
        }
        if (pos == -1)
            return {-1, -1};

        int p1 = pos, p2 = pos; // 搜索边界
        while (p1 >= 0) {
            if (nums[p1] == target) {
                p1--;
            } else
                break;
        }
        while (p2 < nums.size()) {
            if (nums[p2] == target) {
                p2++;
            } else
                break;
        }
        return {p1+1, p2-1};
    }
};
```

现在看看二分难吗？是不是很简单，套路基本就是搜，搜到了处理一下，搜不到再处理一下。

## 总结：二分查找及其应用

这篇文章详细讲解了二分查找的基本原理、不同变体的使用方法及其在几道经典题目中的应用。以下是对文章内容的总结：

#### 一、二分查找的基本原理

二分查找是一种高效的查找算法，适用于已经排序的数组。其核心思想是每次将查找范围缩小一半，直到找到目标元素或范围为空。其时间复杂度为 O(log n)。

#### 二、开区间与闭区间

1. **开区间 `[left, right)`**：
   - 查找范围是 `[left, right)`，即 `right` 不包含在查找范围内。
   - 退出循环条件为 `while (left < right)`，防止越界。
   - 适用于需要严格防止越界的场景。

2. **闭区间 `[left, right]`**：
   - 查找范围是 `[left, right]`，即 `right` 包含在查找范围内。
   - 退出循环条件为 `while (left <= right)`，更直观且不会越界。
   - 适用于大多数查找问题。

#### 三、应用实例

1. **704. 二分查找**：
   - 题目：在排序数组中查找目标值，返回其索引，找不到返回 -1。
   - 思路：可以使用开区间或闭区间的方式进行查找，本文推荐闭区间 `[left, right]` 的方式，便于处理边界条件。

2. **35. 搜索插入位置**：
   - 题目：在排序数组中查找目标值的插入位置。
   - 思路：在查找过程中，如果找不到目标值，返回 `left`，因为 `left` 最终会指向第一个大于等于目标值的位置。

3. **744. 寻找比目标字母大的最小字母**：
   - 题目：在排序字符数组中找到大于目标字符的最小字符。
   - 思路：如果目标字符存在，则返回其下一个不相同的字符；否则，返回 `left` 指向的字符，注意处理越界情况。

4. **34. 在排序数组中查找元素的第一个和最后一个位置**：
   - 题目：在排序数组中查找目标值的第一个和最后一个位置。
   - 思路：先用二分查找找到一个目标值的位置，然后从该位置向两边扩展，找到目标值的边界。注意在退出循环时，边界值需要处理准确。

### 核心思想

1. **选择区间类型**：根据题目要求和边界条件，选择合适的区间类型（开区间或闭区间）。
2. **处理边界条件**：在编写循环条件和更新索引时，要小心处理边界情况，防止越界。
3. **扩展查找结果**：在找到目标值后，根据题目要求，可能需要进一步处理，如查找边界或插入位置。
4. **优化性能**：二分查找的效率在于快速缩小查找范围，确保算法的时间复杂度为 O(log n)。




