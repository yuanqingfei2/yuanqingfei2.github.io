---
title: Leetcode Day8 -- 动态规划2
date: "2022-01-27T04:39:00.000Z"
---

## 连续子数组的最大和

>输入一个整型数组，数组中的一个或连续多个整数组成一个子数组。求所有子数组的和的最大值。要求时间复杂度为O(n)。

关键是连续的，既然要求时间O(n)，只能扫描一遍。不能采用O(n^2)的暴力法。想了一个用队列的办法，但是不对，如下:

```java
class Solution {
    public int maxSubArray(int[] nums) {
        if(nums == null) return 0;
        int max = 0;
        Queue<Integer> result = new LinkedList<>();
        for(int i = 0; i < nums.length; i ++){
            if(result.peek() == null){
                max = nums[i];
                result.offer(nums[i]);
            } else if(nums[i] >= result.peek()){
                result.offer(nums[i]);
                max = max + nums[i];
                max = max - result.poll();
            } else {
                max = 0;
                result = new LinkedList<>();
            }
        }
        System.out.println(result);
        return max;
    }
}
```

标准答案如下，令人不可思议地简单/优美！

关键就是利用动态规划公式：

f(i)=max{f(i−1)+nums[i],nums[i]}

```java
class Solution {
    public int maxSubArray(int[] nums) {
        int pre = 0, maxAns = nums[0];
        for (int x : nums) {
            pre = Math.max(pre + x, x);
            maxAns = Math.max(maxAns, pre);
        }
        return maxAns;
    }
}
```
时间O(n)，空间O(1)。意思倒也明确，加上这个数后，得到的值与这个数相比如果还小，以前的全部舍弃，取这个数，否则就取加和。这样每步的最大值就出来了，然后在同样的遍历中找出一个最大的最大值。

## 礼物的最大价值

>在一个 m*n 的棋盘的每一格都放有一个礼物，每个礼物都有一定的价值（价值大于 0）。你可以从棋盘的左上角开始拿格子里的礼物，并每次向右或者向下移动一格、直到到达棋盘的右下角。给定一个棋盘及其上面的礼物的价值，请计算你最多能拿到多少价值的礼物？

关键是这个公式

f(i,j)=max[f(i,j−1),f(i−1,j)]+grid(i,j)

```java
class Solution {
    public int maxValue(int[][] grid) {
        int m = grid.length, n = grid[0].length;
        for(int i = 0; i < m; i++) {
            for(int j = 0; j < n; j++) {
                if(i == 0 && j == 0) continue;
                if(i == 0) grid[i][j] += grid[i][j - 1] ;
                else if(j == 0) grid[i][j] += grid[i - 1][j];
                else grid[i][j] += Math.max(grid[i][j - 1], grid[i - 1][j]);
            }
        }
        return grid[m - 1][n - 1];
    }
}
```

或者改进版：

```java
class Solution {
    public int maxValue(int[][] grid) {
        int m = grid.length, n = grid[0].length;
        for(int j = 1; j < n; j++) // 初始化第一行
            grid[0][j] += grid[0][j - 1];
        for(int i = 1; i < m; i++) // 初始化第一列
            grid[i][0] += grid[i - 1][0];
        for(int i = 1; i < m; i++)
            for(int j = 1; j < n; j++) 
                grid[i][j] += Math.max(grid[i][j - 1], grid[i - 1][j]);
        return grid[m - 1][n - 1];
    }
}
```
时间O(MN)，空间O(1)

有人更写出更简洁代码，代价是一些空间，真是赞叹！如下：

```java
class Solution {
    public int maxValue(int[][] grid) {
        int row = grid.length;
        int column = grid[0].length;
        //dp[i][j]表示从grid[0][0]到grid[i - 1][j - 1]时的最大价值
        int[][] dp = new int[row + 1][column + 1];
        for (int i = 1; i <= row; i++) {
            for (int j = 1; j <= column; j++) {
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]) + grid[i - 1][j - 1];
            }
        }
        return dp[row][column];
    }
}
```

时间O(MN)，空间O(MN)