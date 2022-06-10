# 【算法】树形DP问题套路


## 树形DP套路
### 使用前提
如果题目求解目标是S规则，则求解流程可以定成，以每一个结点为头节点的子树在S规则下的每一个答案，并且答案一定在其中。

### 步骤
1. 以某个节点`x`为头节点的子树中，分析答案有哪些可能性，且这种分析是以`x`的左子树、`x`的右子树和`x`整颗树的角度来考虑可能性的；
2. 根据分析的可能性，列出所有需要的信息；
3. 合并第二步的信息，对左树和右树提出同样的要求，并写出信息结构；
4. 设计递归函数，递归函数是处理以`x`为头节点的情况下的答案，包括：设计basecase、获得左树和右树返回的信息、整合可能性以及返回信息结构这几步。

### 何时该套路为最优解法
- 如果解法**必须**做第三次的信息的强整合（即获得左树和右树的信息，并整合得到整棵树的信息），只能使用该套路得到最优解；
- 如果解法**不必须要**做第三次信息的强整合，那么Morris遍历可以得到最优解。

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

-----

### 题目2：二叉树中的最大路径和
[LeetCode124](https://leetcode-cn.com/problems/binary-tree-maximum-path-sum/)  
题目描述：**路径**被定义为一条从树中任意节点出发，沿父节点-子节点连接，达到任意节点的序列。同一个节点在一条路径序列中 至多出现一次 。该路径**至少包含一个**节点，且不一定经过根节点。**路径和**是路径中各节点值的总和。给你一个二叉树的根节点 `root` ，返回其 **最大路径和** 。  
思路：考虑以`x`节点为根的二叉树的最大距离有这些情况：（**根是否参与**）
1. `x`节点不参与，即最大路径和的路径在左树或右树中；
2. `x`节点参与，即最大路径和的路径包括`x`节点，这种情况下，最大路径和应该为：`root -> val`、以左树根为起点向下的最大路径和（负值舍弃）、以右树根为起点向下的最大路径和（负值舍弃），三者之和；
3. 最终结果应为：`max(max(左树最大路径和, 右树最大路径和), root -> val + 左树根向下最大路径和(舍负) + 右树根向下最大路径和(舍负)`；
4. 需要的信息：左树和右树各自的最大路径和、以根为起点向下的最大路径和。  

代码：  
```cpp
struct ReturnData {
    int _maxpathsum; // 最大路径和
    int _maxrootsum; // 从root开始向下的最大路径和(只能走一边)
    ReturnData(int maxpathsum, int maxrootsum) : _maxpathsum(maxpathsum), _maxrootsum(maxrootsum) {}
};
class Solution {
private:
ReturnData process(TreeNode* root) {
    if (!root)
        return ReturnData(INT_MIN, INT_MIN);
    ReturnData leftRet = process(root -> left);
    ReturnData rightRet = process(root -> right);
    int p1 = leftRet._maxpathsum;   // 左树最大路径和
    int p2 = rightRet._maxpathsum;  // 右树最大路径和
    int p3 = root -> val;           // root -> val + 两个maxrootsum
    if (leftRet._maxrootsum > 0)
        p3 += leftRet._maxrootsum;
    if (rightRet._maxrootsum > 0)
        p3 += rightRet._maxrootsum;
    int maxpathsum = max(p1, max(p2, p3));
    int largerootsum = max(leftRet._maxrootsum, rightRet._maxrootsum);
    int maxrootsum = root -> val;  // 获得maxrootsum
    if (largerootsum > 0)
        maxrootsum += largerootsum;
    return ReturnData(maxpathsum, maxrootsum);
}
public:
    int maxPathSum(TreeNode* root) {
        return process(root)._maxpathsum;
    }
};
```
