# 【算法】Manacher算法 - 原理及实现


## 最长回文子串
[LeetCode5](https://leetcode-cn.com/problems/longest-palindromic-substring/)  
问题描述：给你一个字符串 `s`，找到 `s` 中最长的回文子串（回文是前后部分对称的字符串，且子串要求连续）。

-----

## 经典算法
中心扩散法：遍历字符串中每一个位置，从该位置出发向两侧扩散寻找回文子串。但是这种方法会忽略掉偶数长度的回文串。解决方法是在每个字符中间加入一个特殊字符`#`（不要求是原串中没出现过的字符），例如`abba`到`#a#b#b#a#`。  
代码：  
```cpp
string longestPalindrome(string s) {
    string tmp = "#", res;
    for (auto ch : s) {
        tmp += ch;
        tmp += "#";
    }
    cout << tmp;
    int index = 0;
    int maxlen = 0;
    for (int i = 0; i < tmp.size(); ++ i) {
        int l = i;
        int r = i;
        while (l >= 0 && r < tmp.size() && tmp[l] == tmp[r]) {
            -- l;
            ++ r;
        }
        ++ l; -- r;
        if (r - l + 1 > maxlen) {
            maxlen = r - l + 1;
            index = l;
        }
    }
    for (int i = index; i < index + maxlen; ++ i) { // 回文的第024...位都是#
        if ((i - index) % 2 != 0)
            res += tmp[i];
    }
    return res;
}
```
时间复杂度：$O\left(n^{2}\right)$  
空间复杂度：$O\left(n\right)$

-----

## Manacher算法
首先明确几个概念：
1. **回文直径**：以某个位置为中心向两边扩展，得到的回文长度为回文直径；
2. **回文半径**：回文直径的一半，向上取整，例如`#a#b#b#a#`，以下标`5`为中心的回文半径为`5`，回文直径为`9`；
3. **回文半径数组**：记录从字符串开头到结尾所有位置为中心的最大回文半径，例如`#a#b#b#a#`的回文半径数组为`[1,2,1,2,5,2,1,2,1]`；
4. **最右回文右边界 R**：初始为`-1`，遍历过程中每得到一个回文子串就更新最右边界，例如`#a#b#b#a#`，第一次更新时（`#`），最右回文右边界为`0`，第二次更新时（`#a#`），最右回文右边界就更新为`2`，第三次更新时（`#`），最右回文右边界还是`2`，没有更大，所以不更新；
5. **最右边界回文的中心点 C**：初始值为`-1`，当最右回文右边界更新时，同步更新该回文串的中心点位置。

Manacher实现过程：  
当遍历到某个中心点`i`时，
1. 该点不在最右回文右边界`R`的左边：从`i`位置的两侧进行暴力的中心扩展；
   ![](/post_images/posts/Coding/Manacher算法/1.jpg "情况1")
2. 该点在最右回文有边界`R`的左边：此时`C`一定在`i`的左边，右边界`R`关于`C`的对称点为`L`就是最右边界回文的左边界，`i`关于`C`的对称点为`i'`，此时一定有这样的图示：`[L..i'..C..i..R]`）。然后**根据`i'`的回文状况**，又分为3类：
   ![](/post_images/posts/Coding/Manacher算法/2.jpg "情况2") 
   1. 以`i'`为中心的回文完全包含在`L~R`中间（开区间）：那么`i`为中心的回文半径和`i'`相同；
      ![](/post_images/posts/Coding/Manacher算法/2_1.jpg "情况2_1")
   2. 以`i'`为中心的回文部分超过`L~R`的范围：（`i'`回文左边界在`L`的左边，右边界不可能超过`R`）那么`i`为中心的回文为`[R'..i..R]`；
      ![](/post_images/posts/Coding/Manacher算法/2_2.jpg "情况2_2")
   3. 以`i'`为中心的回文左边界和`L`相等：`i`为中心的回文长度至少为`L~i'`的长度，是否更长，需要以`i`为中心继续扩展（比较`R'-1 R+1`、`R'-2 R+2`...）。注意，该情况需要把`C`更新为`i`。
      ![](/post_images/posts/Coding/Manacher算法/2_3.jpg "情况2_3")

代码：  
```cpp
string longestPalindrome(string s) {
    string tmp = "#"; // ab -> #a#b#
    for (auto ch : s) {
        tmp += ch;
        tmp += "#";
    }
    vector<int> radius(tmp.size(), 1);
    int R = -1, C = -1;
    int maxIndex = 0;
    for (int i = 0; i < tmp.size(); ++ i) {
        if (i > R) { // 1.
            int L = R = C = i;
            while (L >= 0 && R < tmp.size() && tmp[R] == tmp[L]) {
                -- L; ++ R;
            }
            radius[i] = -- R - i + 1;
            maxIndex = radius[i] > radius[maxIndex] ? i : maxIndex;
        } else { // 2.
            int isym = C - (i - C), L = C - (R - C); // i'
            if (isym - (radius[isym] - 1) > L) { // 2.1
                radius[i] = radius[isym];
            } else if (isym - (radius[isym] - 1) < L) { // 2.2
                radius[i] = R - i + 1;
            } else { // 2.3
                int L = i - (R - i); // 新的L
                C = i;
                while (L >= 0 && R < tmp.size() && tmp[R] == tmp[L]) {
                    -- L; ++ R;
                }
                radius[i] = -- R - i + 1;
                maxIndex = radius[i] > radius[maxIndex] ? i : maxIndex;
            }
        }
    }
    string res;
    int start = maxIndex - radius[maxIndex] + 1;
    for (int i =  start; i < maxIndex + radius[maxIndex]; ++ i) {
        if ((i - start) % 2 != 0)
            res += tmp[i];
    }
    return res;
}
```
时间复杂度：$O\left(n\right)$  
空间复杂度：$O\left(n\right)$

