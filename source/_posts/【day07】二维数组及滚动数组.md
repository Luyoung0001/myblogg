---
title: 【day07】二维数组及滚动数组
date: 2023-03-19 22:10:01
tags:
    - leetcode
    - 算法
    - 数据结构
categories: LeetCode
---

<!--more-->

## 题目 1：[杨辉三角](https://leetcode.cn/problems/pascals-triangle/)

> 给定一个非负整数 numRows，生成「杨辉三角」的前 numRows 行。
在「杨辉三角」中，每个数是它左上方和右上方的数的和。

示例 1：
> 输入: numRows = 5
输出: [[1],[1,1],[1,2,1],[1,3,3,1],[1,4,6,4,1]]

思路 1：利用多维数组模拟的方法：

```java
class Solution {
    public List<List<Integer>> generate(int numRows) {
        int[][] ans = new int[numRows][numRows];

        //输入: numRows = 5
        //输出: [[1],[1,1],[1,2,1],[1,3,3,1],[1,4,6,4,1]]

        List<List<Integer>> result = new ArrayList<>();
        ans[0][0] = 1;
        List<Integer> temp1 = new ArrayList<>();
        // 增加第一个元素
        temp1.add(1);
        result.add(temp1);

        for(int i = 1; i < numRows; i++){
            List<Integer> temp = new ArrayList<>();
            for(int j = 0; j <= i; j++){
                if(j == 0 || j == i){
                    ans[i][j] = 1;
                }else{
                    ans[i][j] = ans[i-1][j]+ans[i-1][j-1];
                }
                temp.add(ans[i][j]);
            }
            result.add(temp);
        }
        return result;
    }
}
```

## 题目 2：[杨辉三角 II](https://leetcode.cn/problems/pascals-triangle-ii/)

> 给定一个非负索引 rowIndex，返回「杨辉三角」的第 rowIndex 行。
在「杨辉三角」中，每个数是它左上方和右上方的数的和。

示例 1：
> 输入: rowIndex = 3
输出: [1,3,3,1]

思路：上一个算法已经得到了杨辉三角，这个题目更简单了，直接返回该行就行：

```java
class Solution {
    public List<Integer> getRow(int rowIndex) {
        int numRows = rowIndex+1;
        int[][] ans = new int[numRows][numRows];
        ans[0][0] = 1;
        List<Integer> temp = new ArrayList<>();
        temp.add(1);
        for (int i = 1; i < numRows; i++) {
            temp.clear();
            for (int j = 0; j <= i; j++) {
                if (j == 0 || j == i) {
                    ans[i][j] = 1;
                } else {
                    ans[i][j] = ans[i - 1][j] + ans[i - 1][j - 1];
                }
                if (i == numRows - 1) temp.add(ans[i][j]);
            }
        }
        return temp;
    }
}
```

## 题目 3：[图片平滑器](https://leetcode.cn/problems/image-smoother/)
> 图像平滑器 是大小为 3 x 3 的过滤器，用于对图像的每个单元格平滑处理，平滑处理后单元格的值为该单元格的平均灰度。
每个单元格的  平均灰度 定义为：该单元格自身及其周围的 8 个单元格的平均值，结果需向下取整。（即，需要计算蓝色平滑器中 9 个单元格的平均值）。
如果一个单元格周围存在单元格缺失的情况，则计算平均灰度时不考虑缺失的单元格（即，需要计算红色平滑器中 4 个单元格的平均值）。

示例 1：

> 输入:img = [[1,1,1],[1,0,1],[1,1,1]]
输出:[[0, 0, 0],[0, 0, 0], [0, 0, 0]]
解释:
对于点 (0,0), (0,2), (2,0), (2,2): 平均(3/4) = 平均(0.75) = 0
对于点 (0,1), (1,0), (1,2), (2,1): 平均(5/6) = 平均(0.83333333) = 0
对于点 (1,1): 平均(8/9) = 平均(0.88888889) = 0

思路：遍历计算

```java
class Solution {
    public int[][] imageSmoother(int[][] img) {
        int m = img.length, n = img[0].length;
        int[][] ret = new int[m][n];
        for (int i = 0; i < m; i++) {
            for (int j = 0; j < n; j++) {
                int num = 0, sum = 0;
                for (int x = i - 1; x <= i + 1; x++) {
                    for (int y = j - 1; y <= j + 1; y++) {
                        if (x >= 0 && x < m && y >= 0 && y < n) {
                            num++;
                            sum += img[x][y];
                        }
                    }
                }
                ret[i][j] = sum / num;
            }
        }
        return ret;
    }
}
```

## 题目 4：[范围求和 II](https://leetcode.cn/problems/range-addition-ii/)
> 给你一个 m x n 的矩阵 M ，初始化时所有的 0 和一个操作数组 op ，其中 ops[i] = [ai, bi] 意味着当所有的 0 <= x < ai 和 0 <= y < bi 时， M[x][y] 应该加 1。
在 执行完所有操作后 ，计算并返回 矩阵中最大整数的个数 。

示例 1：
> 输入: m = 3, n = 3，ops = [[2,2],[3,3]]
输出: 4
解释: M 中最大的整数是 2, 而且 M 中有4个值为2的元素。因此返回 4。

思路：无：

```java
class Solution {
    public int maxCount(int m, int n, int[][] ops) {
        int mina = m, minb = n;
        for (int[] op : ops) {
            mina = Math.min(mina, op[0]);
            minb = Math.min(minb, op[1]);
        }
        return mina * minb;
    }
}
```

## 题目 5：[甲板上的战舰](https://leetcode.cn/problems/battleships-in-a-board/)
> 给你一个大小为 m x n 的矩阵 board 表示甲板，其中，每个单元格可以是一艘战舰 'X' 或者是一个空位 '.' ，返回在甲板 board 上放置的 战舰 的数量。
战舰 只能水平或者垂直放置在 board 上。换句话说，战舰只能按 1 x k（1 行，k 列）或 k x 1（k 行，1 列）的形状建造，其中 k 可以是任意大小。两艘战舰之间至少有一个水平或垂直的空位分隔 （即没有相邻的战舰）。

示例 1：
>输入：board = [["X",".",".","X"],[".",".",".","X"],[".",".",".","X"][".",".",".","."]]
输出：2

思路 1：

```java
class Solution {
    public int countBattleships(char[][] board) {
        int row = board.length;
        int col = board[0].length;
        int ans = 0;
        for (int i = 0; i < row; ++i) {
            for (int j = 0; j < col; ++j) {
                if (board[i][j] == 'X') {
                    board[i][j] = '.';
                    for (int k = j + 1; k < col && board[i][k] == 'X'; ++k) {
                        board[i][k] = '.';
                    }
                    for (int k = i + 1; k < row && board[k][j] == 'X'; ++k) {
                        board[k][j] = '.';
                    }
                    ans++;
                }
            }
        }
        return ans;
    }
}
```

今天的题目都好简单~
前几天生病了，断更了好几天。但是今天恢复了，不会在断更了！

