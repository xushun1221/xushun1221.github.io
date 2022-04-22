# 【算法】二叉树的Morris遍历


## Morris遍历是什么
一种遍历二叉树的方式，且时间复杂度为$O\left(n\right)$，空间复杂度为$O\left(1\right)$，利用原树中叶节点大量空闲指针，来降低空间复杂度。（线索二叉树）

-----

## Morris遍历的实现
### 算法流程
1. 开始时，指针`cur`来到根节点;
2. 如果`cur`没有左孩子，`cur`向右移动，`cur = cur -> right`；
3. 如果`cur`有左孩子，找到左树上最右的节点`mostRight`
   1. 如果`mostRight`的右指针为空，让其指向`cur`，然后`cur`向左移动，`cur = cur -> left`；
   2. 如果`mostRight`右指针为`cur`，令其为空，然后`cur`向右移动，`cur = cur -> right`；
4. `cur`为空时遍历停止。

所有有左子树的节点都会被`cur`指针访问两次，第二次访问是靠左子树的最右节点的右指针来保存位置的。当`cur`第二次访问到某个节点时，说明该节点的左子树已经被完全遍历了，此时可以向右子树走。

整个过程只使用两个指针，空间复杂度为$O\left(1\right)$。  
遍历左子树的右边界这件事，时间复杂度为$O\left(n\right)$。

### 算法实现
#### Morris序
代码：  
```cpp
void morris_traversal(TreeNode* root) {
    if (!root)
        return;
    TreeNode* cur = root;
    TreeNode* mostRight = nullptr;
    while (cur) {
        // 打印每次cur访问的节点
        cout << cur -> val << " ";
        mostRight = cur -> left;
        if (mostRight) { // 有左子树
            while (mostRight -> right && mostRight -> right != cur)
                mostRight = mostRight -> right;
            if (!mostRight -> right) {
                mostRight -> right = cur;
                cur = cur -> left;
                continue;
            } else { // mostRight -> right == cur
                mostRight -> right = nullptr;
            }
        }
        cur = cur -> right;
    }
}
```
时间复杂度：$O\left(n\right)$  
空间复杂度：$O\left(1\right)$

#### 先序
过程：
- 只访问一次的节点，直接打印
- 访问两次的节点，第一次访问时打印

代码：  
```cpp
void morris_pre_traversal(TreeNode* root) {
    if (!root)
        return;
    TreeNode* cur = root;
    TreeNode* mostRight = nullptr;
    while (cur) {
        mostRight = cur -> left;
        if (mostRight) { // 有左子树
            while (mostRight -> right && mostRight -> right != cur)
                mostRight = mostRight -> right;
            if (!mostRight -> right) {
                cout << cur -> val << " "; // 第一次访问时打印
                mostRight -> right = cur;
                cur = cur -> left;
                continue;
            } else { // mostRight -> right == cur
                mostRight -> right = nullptr;
            }
        } else { // 没有左子树的节点只访问一次 直接打印
            cout << cur -> val << " ";
        }
        cur = cur -> right;
    }
}
```

#### 中序
过程：
- 只访问一次的节点，直接打印
- 访问两次的节点，第二次访问时打印

代码：  
```cpp
void morris_in_traversal(TreeNode* root) {
    if (!root)
        return;
    TreeNode* cur = root;
    TreeNode* mostRight = nullptr;
    while (cur) {
        mostRight = cur -> left;
        if (mostRight) { // 有左子树
            while (mostRight -> right && mostRight -> right != cur)
                mostRight = mostRight -> right;
            if (!mostRight -> right) {
                mostRight -> right = cur;
                cur = cur -> left;
                continue;
            } else { // mostRight -> right == cur
                mostRight -> right = nullptr;
            }
        }
        // 能访问两次的节点 第一次访问时continue了不会来到这 第二次会来到这句进行打印
        // 只能访问一次的节点直接打印
        cout << cur -> val << " ";
        cur = cur -> right;
    }
}
```

#### 后序
过程：
- 只访问一次的节点，跳过
- 访问两次的节点，第二次访问时，**逆序**打印它的左子树的右边界
- 最后，**逆序**打印整棵树的右边界

代码：  
```cpp
// 将一颗树的右边界逆序
TreeNode* reverseRightEdge(TreeNode* root) {
    TreeNode* pre = nullptr;
    TreeNode* next = nullptr;
    while (root) {
        next = root -> right;
        root -> right = pre;
        pre = root;
        root = next;
    }
    return pre;
}
// 逆序打印一颗树的右边界
void printRightEdge(TreeNode* root) {
    TreeNode* rear = reverseRightEdge(root);
    TreeNode* cur = rear;
    while (cur) {
        cout << cur -> val << " ";
        cur = cur -> right;
    }
    reverseRightEdge(rear);
}
void morris_post_traversal(TreeNode* root) {
    if (!root)
        return;
    TreeNode* cur = root;
    TreeNode* mostRight = nullptr;
    while (cur) {
        mostRight = cur -> left;
        if (mostRight) { // 有左子树
            while (mostRight -> right && mostRight -> right != cur)
                mostRight = mostRight -> right;
            if (!mostRight -> right) {
                mostRight -> right = cur;
                cur = cur -> left;
                continue;
            } else { // mostRight -> right == cur
                mostRight -> right = nullptr; // 先置空
                printRightEdge(cur -> left); // 第二次访问到时 打印左子树的右边界
            }
        }
        cur = cur -> right;
    }
    printRightEdge(root);
}
```

-----

## 何时Morris遍历为最优解法
- 如果解法**必须**做第三次的信息的强整合（即获得左树和右树的信息，并整合得到整棵树的信息），只能使用该套路得到最优解；
- 如果解法**不必须要**做第三次信息的强整合，那么Morris遍历可以得到最优解。
