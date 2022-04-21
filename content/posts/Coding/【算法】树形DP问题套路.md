---
title: 【算法】树形DP问题套路
date: 2022-04-21
tags: [算法与数据结构, C++]
categories: [Coding]
---

## 树形DP套路
### 使用前提
如果题目求解目标是S规则，则求解流程可以定成，以每一个结点为头节点的子树在S规则下的每一个答案，并且答案一定在其中。

### 步骤
1. 以某个节点`x`为头节点的子树中，分析答案有哪些可能性，且这种分析是以`x`的左子树、`x`的右子树和`x`整颗树的角度来考虑可能性的；
2. 根据分析的可能性，列出所有需要的信息；
3. 合并第二步的信息，对左树和右树提出同样的要求，并写出信息结构；
4. 设计递归函数，递归函数是处理以`x`为头节点的情况下的答案，包括：设计basecase、获得左树和右树返回的信息、整合可能性以及返回信息结构这几步。

-----

## 相关题目
### 题目1：二叉树结点间的最大距离问题
[LeetCode543](https://leetcode-cn.com/problems/diameter-of-binary-tree/)  
题目描述：从二叉树的节点`a`出发，可以向上或向下走，但沿途的节点只能经过一次，到达结点`b`时路径上的节点个数叫做`a`到`b`的距离（包括起始和终点），二叉树任何两个节点之间都有距离，求整颗树上的最大距离。  
思路：考虑以`x`节点为根的二叉树的最大距离有这些情况：（**根是否参与**）
1. `x`不参与，最大距离路径不经过`x`，那么整颗树的最大距离可能来自`x`的**左树的最大距离**，或者**右树的最大距离**；
2. `x`参与，也就是`x`作为整棵树最大距离的路径上的一个节点，那么最大距离的路径就是：左树离`x`最远的节点到右树离`x`最远的节点的距离，也就是左树和右树的高度之和加`1`；
3. 最后结果就是：`max(左树的最大距离, 右树的最大距离, 左树高+右树高+1)`；
4. 分析需要的信息：左树需要提供最大距离和高度，右树相同，那返回信息`x`需要的返回信息就是最大距离和高度。

代码：  
```cpp
struct ReturnData {
    int _maxdist;
    int _height;
    ReturnData(int maxdist, int height) : _maxdist(maxdist), _height(height) {}
};
class Solution {
private:
    ReturnData process(TreeNode* root) {
        // 返回以root为根的树的最大距离和高度
        if (!root)
            return ReturnData(0, 0);
        ReturnData leftRet = process(root -> left);
        ReturnData rightRet = process(root -> right);
        int p1 = leftRet._maxdist;
        int p2 = rightRet._maxdist;
        int p3 = leftRet._height + 1 + rightRet._height;
        int height = max(leftRet._height, rightRet._height) + 1;
        int maxdist = max(p1, max(p2, p3));
        return ReturnData(maxdist, height);
    }
public:
    int diameterOfBinaryTree(TreeNode* root) {
        return process(root)._maxdist - 1;
    }
};
```
