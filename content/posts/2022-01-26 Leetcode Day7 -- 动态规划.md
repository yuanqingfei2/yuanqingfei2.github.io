---
title: Leetcode Day7 -- 动态规划
date: "2022-01-26T09:04:00.000Z"
---

## Fibonacci数列


```java
class Solution {
    public int fib(int n) {
        if(n == 0) return 0;
        else if(n == 1) return 1;
        else {
            int result = fib(n-2)+fib(n-1);
            return result % 1000000007;
        }
    }
}
```
递归的方式效率不行，不能通过所有测试（小丑竟是我自己！）不要用递归。

```java
class Solution {
    public int fib(int n) {
        if(n == 0 || n == 1) return n;
        int i = 0, j = 1, fn = 0;
        for(int k = 2; k <= n; k ++){
            fn = (i + j) % 1000000007;
            i = j;
            j = fn;
        }
        return fn;
        // return (fn % 1000000007);
    }
}
```

为什么上面这个不行？原因在于取模要在循环里面取。

标准答案如下：

动态数组法。 空间O(1)，时间O(n)
```java
class Solution {
    public int fib(int n) {
        final int MOD = 1000000007;
        if (n < 2) {
            return n;
        }
        int p = 0, q = 0, r = 1;
        for (int i = 2; i <= n; ++i) {
            p = q; 
            q = r; 
            r = (p + q) % MOD;
        }
        return r;
    }
}
```

## 青蛙跳台阶

>一只青蛙一次可以跳上1级台阶，也可以跳上2级台阶。求该青蛙跳上一个 n 级的台阶总共有多少种跳法。

应该也是一个Fibonacci数列。所以答案同上。直觉如此，数学推理如下：

设跳上 n 级台阶有 f(n) 种跳法。在所有跳法中，青蛙的最后一步只有两种情况： 跳上 1 级或 2 级台阶。
当为 1 级台阶： 剩 n-1 个台阶，此情况共有 f(n-1) 种跳法；
当为 2 级台阶： 剩 n-2 个台阶，此情况共有 f(n-2) 种跳法。

唯一的区别是f(0) = 1 and f(1) = 1

## 股票利润最大化

>假设把某股票的价格按照时间先后顺序存储在数组中，请问买卖该股票一次可能获得的最大利润是多少？

思路应该是随着下标的增大，记录目前的最小值，以及下标比最小值下标大的最大值。完成扫描后求差。后来想了下，这个思路不对，因为有可能发现最大之后的最小值是不能使用的。

所以笨办法就是对每一个数字都求其所有以后的数字与它之差，这样最大的差就是最大利润。

```java
class Solution {
    public int maxProfit(int[] prices) {
        if(prices == null || prices.length == 0) return 0;
        int result = 0;
        for(int i = 0; i < prices.length; i ++){
            int tempDiff = 0;
            for(int j = i + 1; j < prices.length; j ++){
                if(prices[j] - prices[i] > result) result = prices[j] - prices[i];
            }
        }
        return result;
   }
}
```
时间O(n^2)，空间O(1)

但是更好的答案是一次遍历法， 如下：

```java
public class Solution {
    public int maxProfit(int prices[]) {
        int minprice = Integer.MAX_VALUE;
        int maxprofit = 0;
        for (int i = 0; i < prices.length; i++) {
            if (prices[i] < minprice) {
                minprice = prices[i];
            } else if (prices[i] - minprice > maxprofit) {
                maxprofit = prices[i] - minprice;
            }
        }
        return maxprofit;
    }
}
```

时间O(n), 空间O(1).

