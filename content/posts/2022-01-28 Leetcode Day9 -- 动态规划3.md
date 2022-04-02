---
title: Leetcode Day9 -- 动态规划3
date: "2022-01-28T10:52:00.000Z"
---

## 把数字翻译成字符串

>给定一个数字，我们按照如下规则把它翻译为字符串：0 翻译成 “a” ，1 翻译成 “b”，……，11 翻译成 “l”，……，25 翻译成 “z”。一个数字可能有多个翻译。请编程实现一个函数，用来计算一个数字有多少种不同的翻译方法。

不会。官方解法：

转移方程：f(i)=f(i−1)+f(i−2)[i−1≥0,10≤x≤25]

```java

class Solution {
    public int translateNum(int num) {
        String src = String.valueOf(num);
        int p = 0, q = 0, r = 1;
        for (int i = 0; i < src.length(); ++i) {
            p = q; 
            q = r; 
            r = 0;
            r += q;
            if (i == 0) {
                continue;
            }
            String pre = src.substring(i - 1, i + 1);
            if (pre.compareTo("25") <= 0 && pre.compareTo("10") >= 0) {
                r += p;
            }
        }
        return r;
    }
}
```

时间是O(logn)，空间O(logn)。上面的解释不是很好懂。

有人提示说，本质上就是青蛙跳台，只有能不能跳两级要看条件。如果这样就容易理解多了。

```java
class Solution {
    public static int translateNum(int num) {
        char[] ch = String.valueOf(num).toCharArray();
        int len = ch.length;
        int[] dp = new int[len + 1];
        dp[0] = 1;
        dp[1] = 1;
        for(int i = 2; i <= len; i++){
            int n = (ch[i - 2] - '0') * 10 + (ch[i - 1] - '0');
            if(n >= 10 && n <= 25){
                dp[i] = dp[i - 1] + dp[i - 2];
            }else{
                dp[i] = dp[i - 1];
            }
        }
        return dp[len];
    }
}
```

##  最长不含重复字符的子字符串

>请从字符串中找出一个最长的不包含重复字符的子字符串，计算该最长子字符串的长度
