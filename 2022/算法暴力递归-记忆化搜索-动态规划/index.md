# 【算法】暴力递归->记忆化搜索->动态规划


## 三者关系
1. 找到某个解决问题的尝试方法
2. **暴力递归**就是不经过优化的暴力尝试
3. **记忆化搜索**的动态规划，在暴力递归基础上优化，无需规定每个状态的依赖关系
4. **严格表结构**的动态规划，必须将每一个状态的依赖关系规定好
5. 在某些问题上，记忆化搜索和严格表结构的动态规划时间复杂度相同
6. 根据位置依赖关系有可能进行进一步优化得到**精致的严格表结构**的动态规划
7. 最关键的步骤是如何选择一个比较好的尝试方法

-----

## 相关题目
### 题目1：机器人的路径总数
题目描述：给定一个常数`n`，定义一段`1..n`的路，给定起始位置`start`（1..n），和目标位置`end`（1..n），规定机器人一定需要走`step`步（每次走一格，可以向左或向右），找出机器人从起始位置到目标位置共有多少种走法。  

#### 尝试 - 暴力递归
1. 当机器人走到某个位置后剩余步数为0，检查是否走到目标位置；
2. 机器人在边上时（1或n位置）只能向中间走；
3. 机器人在中间时，可以向两边走。

代码：  
```cpp
int process(int n, int cur, int end, int rest) {
    // cur  当前位置
    // rest 剩余步数
    if (rest == 0) // 无剩余步数 检查是否到达目标位置
        return cur == end ? 1 : 0;
    if (cur == 1)  // 当前在最左边 只能向右走
        return process(n, cur + 1, end, rest - 1);
    if (cur == n)  // 当前在最右边 只能向左走
        return process(n, cur - 1, end, rest - 1);
    // 在中间位置可以尝试 向左 或 向右
    return process(n, cur - 1, end, rest - 1) + process(n, cur + 1, end, rest - 1);
}
int walk_way(int n, int start, int end, int step) {
    // n     1..n位置
    // start 初始位置
    // end   目标位置
    // step  要走的步数
    return process(n, start, end, step);
}
```
时间复杂度：$O\left(2^{step}\right)$ 递归过程为一棵高度为k的二叉树

#### 分析尝试方法
尝试过程通过递归函数`process(int n, int cur, int end, int rest)`来实现，其中`n  end`是常量，该过程的返回值完全由`cur  rest`两个变量决定，例如，`cur = 2, rest = 4`，机器人当前在2位置，剩余步数为4，假设返回值为4，在递归过程中可能多次尝试到该状态`process(cur= 2, rest = 4, ..)`，无论该状态是由什而么状态转移来，只要到达该状态，它的返回值一定为4，这种尝试方法也被称为**无后效性**尝试。如果将递归过程中的到达的状态和其对应的返回值记录下来，当再次遇到该状态时，就无需继续再次进行递归计算，用一定的空间换取了时间复杂度的优化。

#### 优化 - 记忆化搜索
使用一个二维数组记录递归过程中遇到的`process(cur, rest)`的返回值，根据暴力递归的代码，可以看出`1 <= cur <= n`，`0 <= rest <= step`，所以只需$n\times\left(step+1\right)$大小的二维数组即可。（使用哈希表之类的结构也可以）

代码：  
```cpp
int process_1(int n, int cur, int end, int rest, vector<vector<int>>& dp) {
    if (dp[cur][rest] != -1)
        return dp[cur][rest];
    if (rest == 0) {
        dp[cur][rest] = cur == end ? 1 : 0;
        return dp[cur][rest];
    }
    if (cur == 1)
        dp[cur][rest] = process_1(n, cur + 1, end, rest - 1, dp);
    else if (cur == n)
        dp[cur][rest] = process_1(n, cur - 1, end, rest - 1, dp);
    else
        dp[cur][rest] = process_1(n, cur - 1, end, rest - 1, dp) + process_1(n, cur + 1, end, rest - 1, dp);
    return dp[cur][rest];
}
int walk_way_1(int n, int start, int end, int step) {
    vector<vector<int>> dp(n + 1, vector<int>(step + 1, -1));
    return process_1(n, start, end, step, dp);
}
```
时间复杂度：$O\left(step\times n\right)$ 对dp数组而言，求解每一个单元的时间代价为$O\left(1\right)$（如果左右的单元已经求解过）

#### 优化 - 严格表结构的动态规划
