---
title: Leetcode Day4 -- 查找2
date: "2022-01-23T04:08:00.000Z"
---

## 在二维数组中查找

```java
class Solution {
    public boolean findNumberIn2DArray(int[][] matrix, int target) {
        if(matrix == null || matrix.length == 0 || matrix[0].length == 0) return false;
        int i = findIndex(matrix[0], target);
        int col[] = new int[matrix.length];
        for(int k = 0; k < col.length; k++ ){
            col[k] = matrix[k][i];
            System.out.println("col: " + col[k]);
        }
        int j = findIndex(col, target);
        System.out.println(matrix[j][i]);
        if(matrix[j][i] == target) return true;
        else return false;
    }

    private int findIndex(int[] rowcol, int target){
        int i = 0, j = rowcol.length -1;
        while(i < j){
            int mid = i + (j - i)/2;
            if(rowcol[mid] > target) j = mid;
            else i = mid + 1;
        }
        System.out.println(i);
        System.out.println(j);
        if(rowcol[i] == target) return i;
        else return i-1 >= 0 ? i - 1 : 0;
    }
}
```

思路也很简单，因为行和列都是增序的。先从第一行中用二分法查找列的下标，找到后，再用二分法在那一列中查找行标。然后比较即可。但是上面的解法不能通过所有测试用例（白瞎我一个多小时）。标准答案如下：

```java
class Solution {
    public boolean findNumberIn2DArray(int[][] matrix, int target) {
        if (matrix == null || matrix.length == 0 || matrix[0].length == 0) {
            return false;
        }
        int rows = matrix.length, columns = matrix[0].length;
        int row = 0, column = columns - 1;
        while (row < rows && column >= 0) {
            int num = matrix[row][column];
            if (num == target) {
                return true;
            } else if (num > target) {
                column--;
            } else {
                row++;
            }
        }
        return false;
    }
}
```

这个思路就是如果这个数字大于目标值，向左移(columnn --)，如果这个数字小于目标值，则向下移(row ++)，这样一个负责行，一个负责列，来最终搜索到，其实也可以选择，如果目标大时上移(row --)，小时右移(column ++)。如下：

```java
class Solution {
    public boolean findNumberIn2DArray(int[][] matrix, int target) {
        if (matrix == null || matrix.length == 0 || matrix[0].length == 0) {
            return false;
        }
        int rows = matrix.length, columns = matrix[0].length;
        int row = rows - 1, column = 0;
        while (column < columns && row >= 0) {
            int num = matrix[row][column];
            if (num == target) {
                return true;
            } else if (num > target) {
                row --;
            } else {
                column ++;
            }
        }
        return false;
    }
}
```
时间O(n+m)，空间O(1)

最后所谓暴力查找，不提也罢。

## 旋转升序数组中的最小数字

既然原来是升序的，现在不再是了，那么只要找到第一个变小的数字的下标[i]。
```java
class Solution {
    public int minArray(int[] numbers) {
        if(numbers == null || numbers.length == 0) return -1;
        for(int i = 0; i < numbers.length; i ++){
            if(i + 1 < numbers.length && numbers[i + 1] < numbers[i]) return numbers[i + 1];
        }
        return numbers[0];
    }
}
```

时间O(n)，空间O(1)

标准答案采用二分法，**二分法取左中**，如下：
```java
class Solution {
    public int minArray(int[] numbers) {
        int low = 0;
        int high = numbers.length - 1;
        while (low < high) {
            int pivot = low + (high - low) / 2;
            if (numbers[pivot] < numbers[high]) {
                high = pivot;
            } else if (numbers[pivot] > numbers[high]) {
                low = pivot + 1;
            } else {
                high -= 1;
            }
        }
        return numbers[low];
    }
}
```

时间O(logn)(最糟糕情况O(n))，空间O(1) 

## 第一个只出现一次的字符

```java
class Solution {
    public char firstUniqChar(String s) {
        if(s == null || s.length() == 0) return ' ';
        int[] counts = new int[26];
        for(int i = 0; i < s.length(); i ++)
            counts[s.charAt(i) - 'a'] ++;
        for(int i = 0; i < s.length(); i ++)
            if(counts[s.charAt(i) - 'a'] == 1) return s.charAt(i);

        return ' ';
    }
}
```

就是扫描两遍，第一遍算次数，第二遍查看次数。

时间O(n)，空间O(1)

官方答案赶紧还不如我的简单，如下：
```java
class Solution {
    public char firstUniqChar(String s) {
        Map<Character, Integer> frequency = new HashMap<Character, Integer>();
        for (int i = 0; i < s.length(); ++i) {
            char ch = s.charAt(i);
            frequency.put(ch, frequency.getOrDefault(ch, 0) + 1);
        }
        for (int i = 0; i < s.length(); ++i) {
            if (frequency.get(s.charAt(i)) == 1) {
                return s.charAt(i);
            }
        }
        return ' ';
    }
}
```