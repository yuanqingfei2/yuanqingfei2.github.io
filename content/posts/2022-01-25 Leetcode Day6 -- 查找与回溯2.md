---
title: Leetcode Day6 -- 查找与回溯2
date: "2022-01-25T05:18:00.000Z"
---

## 判定一个树是不是另外一个树的子结构

递归加先序遍历

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    public boolean isSubStructure(TreeNode A, TreeNode B) {
        return (A != null && B != null) && (recur(A, B) || isSubStructure(A.left, B) || isSubStructure(A.right, B));
    }
    boolean recur(TreeNode A, TreeNode B) {
        if(B == null) return true;
        if(A == null || A.val != B.val) return false;
        return recur(A.left, B.left) && recur(A.right, B.right);
    }
}
```

时间O(MN)，空间O(M)

## 打印二叉树的镜像

其实就是把所有的左右子树对调。依然用递归

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    public TreeNode mirrorTree(TreeNode root) {
        if(root == null) return null;
        TreeNode temp = root.left;
        root.left = mirrorTree(root.right);
        root.right = mirrorTree(temp);
        return root;
    }
}

时间O(n), 空间O(n)

```

## 判断一个二叉树是不是对称

思路是首先得到镜像树，然后比较这两颗树

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    public boolean isSymmetric(TreeNode root) {
        TreeNode mirror = mirrorTree(root);
        return compareTree(root, mirror);
    }

    private TreeNode mirrorTree(TreeNode root) {
        if(root == null) return null;
        // 这里要新建一个
        TreeNode newRoot = new TreeNode(root.val);
        TreeNode temp = root.left;
        newRoot.left = mirrorTree(root.right);
        newRoot.right = mirrorTree(temp);
        return newRoot;
    }

    private boolean compareTree(TreeNode A, TreeNode B){
        if(B == null) return true;
        if(A == null || A.val != B.val) return false;
        return compareTree(A.left, B.left) && compareTree(A.right, B.right);
    }
}
```

上面的不对，暂时还不知道为什么。后来网上看到原来[很多人也是这么想的](https://leetcode-cn.com/problems/dui-cheng-de-er-cha-shu-lcof/solution/yi-zhi-wei-truede-pen-you-ke-yi-jin-lai-lcelf/)，问题的关键点在于那个Mirror上，那个mirror树必须是一个新的树，而不能在老树上弄。如上注释。这样就可以了。


标准答案如下：

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    public boolean isSymmetric(TreeNode root) {
        if(root == null) return true;
        return compareTree(root.left, root.right);
    }

    private boolean compareTree(TreeNode A, TreeNode B){
        if(B == null && A == null) return true;
        if(A == null || B == null || A.val != B.val) return false;
        return compareTree(A.left, B.right) && compareTree(A.right, B.left);
    }
}
```