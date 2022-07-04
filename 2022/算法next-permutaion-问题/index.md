# 【算法】Next Permutaion 问题


## 问题描述
给定一个数字或字符序列，要求找到它的下一个按字典序升序排列的序列。

相关题目：
- [LeetCode31.下一个排列](https://leetcode.cn/problems/next-permutation/)
- [LeetCode556.下一个更大的元素](https://leetcode.cn/problems/next-greater-element-iii/)
- [LeetCode60.排列序列](https://leetcode.cn/problems/permutation-sequence/)

---

## 解题思路
我们使用两次遍历的方法以`O(n)`的时间复杂度来实现。

下一个序列一定比当前的序列要大(按字典序)，除非已经是字典序最大的序列了，我们要实现的目标是，找到一个比当前序列要大的新序列，但是变大的幅度尽可能的小。实现的方法为：
1. 我们需要找到一个左边的`较小数`，和一个右边的`较大数`，交换这两个数，这就获得了一个更大的新序列；
2. 同时，我们需要新序列仅比当前序列大，所以，这个`较小数`要尽可能靠右，交换完成后，`较大数`右边的所有元素要按照升序重新排列，这能保证变大的幅度尽可能小；
3. 所以，我们需要寻找一个`较小数-较大数`对，这个数对应该为序列从右向左的第一个升序对，寻找数对的方法：
   1. 从右向左寻找第一个升序相邻的数对，这样遍历的方式可以保证，该数对右边的数一定降序，记住当前`较小数`的位置；
   2. 在`较小数`右侧从右向左找第一个大于`较小数`的`较大数`，因为较小数右侧的为降序序列，所以`较大数`为大于`较小数`的最小数，交换这两个数，依然可以保证`原较小数位置`右侧为降序序列；
   3. 将`原较小数位置`右侧序列反转，即可得到升序序列，这样即可保证变大的幅度尽可能小。

---

## 实现
```cpp
void nextPermutation(vector<int>& nums) {
    int smaller = -1, large = -1;
    for (int i = nums.size() - 2; i >= 0; -- i) {
        if (nums[i] < nums[i + 1]) {
            smaller = i;
            break;
        }
    }
    if (smaller == -1) {
        reverse(nums.begin(), nums.end());
        return;
    }
    for (int i = nums.size() - 1; i > smaller; -- i) {
        if (nums[i] > nums[smaller]) {
            swap(nums[i], nums[smaller]);
            break;
        }
    }
    reverse(nums.begin() + smaller + 1, nums.end());
}
```
时间复杂度：$O\left(n\right)$  
空间复杂度：$O\left(1\right)$
