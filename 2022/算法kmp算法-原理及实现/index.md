# 【算法】KMP算法 - 原理及实现


## 字符串匹配算法
[LeetCode28](https://leetcode-cn.com/problems/implement-strstr/)  
问题描述：给你两个字符串`haystack`和`needle`，请你在`haystack`字符串中找出`needle`字符串出现的第一个位置（下标从 `0` 开始）。如果不存在，则返回`-1`。如果`needle`长度为`0`，返回`0`。

-----

## 经典算法
思路：从`haystack`开始位置向后依次尝试匹配`needle`，尝试失败就从下一个位置重新开始。（暴力匹配算法时间复杂度较高）  
代码：  
```cpp
class Solution {
public:
    int strStr(string haystack, string needle) {
        int len1 = haystack.length();
        int len2 = needle.length();
        if (!len2) 
            return 0;
        bool same  = true;
        for (int i = 0; i < len1 - len2 + 1; ++ i){
            same = true;
            for (int j = 0; j < len2; ++ j){
                if (haystack[i + j] != needle[j])
                    same = false;
            }
            if (same)
                return i;
        }
        return -1;
    }
};
```
时间复杂度：$O\left(mn\right)$  
空间复杂度：$O\left(1\right)$

-----

## KMP算法

### 字符串的最长相同前后缀
在实现KMP算法之前，考虑这样一个前置问题：给定一个字符串`str`，找到该字符串最长的相同的前缀和后缀的长度（`0 ~ str.size() - 1`），例如`str == aabstaab`，该字符串最长相同前后缀为`aab`，长度为`3`。

为字符串`str`定义这样一个数组`next`：`next[i]`表示子串`str[0, .. ,i - 1]`的最长相同前后缀长度。  
求解思路：这是一个动态规划过程
1. 对于`str[0]`，之前没有子串，`next[0] = -1`；
2. 对于`str[1]`，子串`str[0]`，前后缀和子串相同，不符合定义，`next[1] = 0`；
3. 当`i >= 2`时，如果`str[i - 1]`和最大前缀的后一个字符相同，那`next[i] = 最大前缀长度+1`，否则`str[i - 1]`要和最大前缀的后一个位置表示的前缀的后一个字符比较……

代码：  
```cpp
vector<int> getNextArray(string needle) {
    // dp
    vector<int> next(needle.size(), 0);
    next[0] = -1;
    int i = 2;
    int cn = 0;
    while (i < next.size()) {
        if (needle[i - 1] == needle[cn]) {
            next[i ++] = ++ cn;
        }
        else if (cn > 0) {
            cn = next[cn];
        }0123
        else {
            next[i ++] = 0;
        }
    }
    return next;
}
```
时间复杂度：$O\left(m\right)$  
空间复杂度：$O\left(1\right)$

### KMP算法实现
思路：KMP算法的核心思想是，利用最长相同前后缀，在`needle`中选择下一个比较的位置。利用`next`数组，将下一个在`needle`中的比较位置，跳回到已经匹配成功的前缀的后一个位置上。  
代码：  
```cpp
int strStr(string haystack, string needle) {
    // KMP
    if (!needle.size())
        return 0;
    if (haystack.size() < needle.size())
        return -1;
    int i1 = 0; // index of haystack
    int i2 = 0; // index of needle
    vector<int> next(getNextArray(needle)); // O(m)
    while (i1 < haystack.size() && i2 < needle.size()) { // O(n)
        if (haystack[i1] == needle[i2]) { // 匹配
            ++ i1;
            ++ i2;
        }
        else if (i2 == 0) { // needle首字符不匹配
            ++ i1;
        }
        else {
            i2 = next[i2]; // 更新匹配位置
        }
    }
    // i2 越界说明匹配到
    return i2 == needle.size() ? i1 - i2 : -1;
}
```
时间复杂度：$O\left(m + n\right)$  
空间复杂度：$O\left(m\right)$ next数组
