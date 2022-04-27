# 【算法】暴力递归->记忆化搜索->动态规划


## 三者关系
1. 找到某个解决问题的尝试方法
2. **暴力递归**就是不经过优化的暴力尝试
3. **记忆化搜索**的动态规划，在暴力递归基础上优化，无需规定每个状态的依赖关系
4. **严格表结构**的动态规划，必须将每一个状态的依赖关系规定好
5. 在某些问题上，记忆化搜索和严格表结构的动态规划时间复杂度相同
6. 根据位置依赖关系有可能进行进一步优化得到**精致的严格表结构**的动态规划
7. 最关键的步骤是如何选择一个比较好的尝试方法

### 从暴力递归到记忆化搜索
加缓存。

### 从记忆化搜索到严格表结构动态规划
1. 确定可变参数的数量和变化范围，如两个可变参数就是二维表；
2. 标出目标位置（要得到的结果）；
3. 标出basecase，即无需计算的初始位置（边界条件）；
4. 确定普通位置的依赖关系；
5. 确定从初始位置到目标位置的依次计算的顺序，依次填表。

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
时间复杂度：$O\left(2^{step}\right)$ 递归过程为一棵高度为`step`的二叉树

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
从上面的分析已经知道，每一次的尝试结果仅依赖于`cur`和`rest`两个值，问题的最终求解目标为`process(cur = start, rest = step)`，这个结果也依赖于它的子问题。定义一个dp数组`dp[1..n][0..step]`，数组的每一个位置表示一个子问题的解，最终求解目标就是求出`dp[start][step]`的值。

如何将dp数组填满，其实就是暴力递归的尝试过程：
1. basecase：当`rest == 0`时，如果`cur == end`，该位置为1，否则为0，在dp数组中体现为，`dp[end][0] = 1, dp[1..end-1, end+1..n][0] = 0`（dp数组的第一列确定了）；
2. 当`cur == 1`时，即机器人在最左边，它向右走（该状态依赖于`cur == 2, rest - 1`），在dp数组中体现为，`dp[1][rest] = dp[2][rest-1]`，即第一行的单元由其对应的第二行的上一个单元决定；
3. 当`cur == n`时，同上，`dp[n][rest] = dp[n-1][rest-1]`；
4. 当机器人在中间位置时，机器人可以向左或向右，即`dp[cur][rest] = dp[cur-1][rest-1] + dp[cur+1][rest-1]`。

代码：  
```cpp
int walk_way_2(int n, int start, int end, int step) {
    vector<vector<int>> dp(n + 1, vector<int>(step + 1, 0));
    dp[end][0] = 1;
    for (int rest = 1; rest <= step; ++ rest) {
        dp[1][rest] = dp[2][rest - 1];
        for (int cur = 2; cur <= n - 1; ++ cur)
            dp[cur][rest] = dp[cur - 1][rest - 1] + dp[cur + 1][rest - 1];
        dp[n][rest] = dp[n - 1][rest - 1];
    }
    return dp[start][step];
}
```
时间复杂度：$O\left(step\times n\right)$

-----

### 题目2：组合面值
题目描述：给定一个正整数数组`arr`，`arr[i]`代表一枚硬币的面值（可能有重复的硬币），给定一个目标`aim`，返回能组合为目标面值的最少的硬币数量。例，`arr[] = {2,7,3,5,3}, aim = 10`，答案为2。

相似题目：  
[LeetCode322](https://leetcode-cn.com/problems/coin-change/)  
[LeetCode518](https://leetcode-cn.com/problems/coin-change-2/)

#### 尝试 - 暴力递归
1. 从左向右依次尝试选择或不选该硬币；
2. 在可以成功组合的情况中选择硬币数少的作为答案即可。

代码：  
```cpp
int process(vector<int>& arr, int rest, int index) {
    // rest  剩下多少面值待组合
    // index 从index往后的硬币可取
    // 返回值 硬币的最少个数
    if (rest < 0)  // 无法组成负数面值
        return -1;
    if (rest == 0) // 组成面值为0 需要0枚硬币
        return 0;
    else if (index == arr.size()) // rest > 0 // 硬币已经选完了 rest还大于0 证明无法组合
        return -1;
    else {
        // rest > 0 也还有硬币
        // 不选 或 选 index硬币
        int p1 = process(arr, rest, index + 1);
        int p2 = process(arr, rest - arr[index], index + 1);
        if (p1 == -1 && p2 == -1) // 两种情况都无法组合
            return -1;
        else {
            if (p1 == -1)   // 不选index无法组合
                return p2 + 1;
            else if (p2 == -1)   // 选index无法组合
                return p1;
            else
                return min(p1, p2 + 1); // 两种情况都可 返回两种情况较小的
        }
    }
}
int min_coins(vector<int>& arr, int aim) {
    // arr 硬币数组
    // aim 目标面值
    return process(arr, aim, 0);
}
```

#### 优化 - 记忆化搜索
可以看出，每个状态由当前选择硬币位置`index`和剩余的面值`rest`决定，所以申请一个二维数组`dp[0..arr.size()][0..aim]`来保存递归过程的状态信息。

代码：  
```cpp
int process_1(vector<int>& arr, int rest, int index, vector<vector<int>>& dp) {
    if (rest < 0)  // 相当于命中无效缓存
        return -1;
    if (dp[index][rest] != -2) // 命中缓存
        return dp[index][rest];
    if (rest == 0)
        dp[index][rest] = 0;
    else if (index == arr.size()) // rest > 0
        dp[index][rest] = -1;
    else {
        int p1 = process_1(arr, rest, index + 1, dp);
        int p2 = process_1(arr, rest - arr[index], index + 1,dp);
        if (p1 == -1 && p2 == -1)
            dp[index][rest] = -1;
        else {
            if (p1 == -1)
                dp[index][rest] = p2 + 1;
            else if (p2 == -1)
                dp[index][rest] = p1;
            else // p1 != -1 && p2 != -1
                dp[index][rest] = min(p1, p2 + 1);
        }
    }
    return dp[index][rest];
}
int min_coins_1(vector<int>& arr, int aim) {
    vector<vector<int>> dp(arr.size() + 1, vector<int>(aim + 1, -2));
    return process_1(arr, aim, 0, dp);
}
```

#### 优化 - 严格表结构的动态规划
定义动态规划数组`dp[0..arr.size()][0..aim]`，最终求解目标为`dp[0][aim]`。

如何将dp数组填满，参考暴力尝试过程：
1. `rest < 0`时，认为`dp[..][rest<0] == -1`；
2. basecase：`rest == 0`时，`dp[0..arr.size()][0] = 0`；
3. `index == arr.size() && rest > 0`时，`dp[arr.size()][1..aim] = -1`；
4. 其余位置的依赖关系，每一个`dp[index][rest]`都依赖于`dp[index+1][rest]`和`dp[index+1][rest-arr[index]]`，即依赖于下一行的两个位置；
5. 对于其他位置，需要从左往右、从下往上按行填满。

代码：  
```cpp
int min_coins_2(vector<int>& arr, int aim) {
    vector<vector<int>> dp(arr.size() + 1, vector<int>(aim + 1, 0)); // rest == 0, dp = 0
    for (int rest = 1; rest <= aim; ++ rest)
        dp[arr.size()][rest] = -1;
    for (int index = arr.size() - 1; index >= 0; -- index) {
        for (int rest = 1; rest <= aim; ++ rest) {
            int p1 = dp[index + 1][rest];
            int p2 = -1;
            if (rest - arr[index] >= 0) // 判定是否越界
                p2 = dp[index + 1][rest - arr[index]];
            if (p1 == -1 && p2 == -1)
                dp[index][rest] = -1;
            else {
                if (p1 == -1)
                    dp[index][rest] = p2 + 1;
                else if (p2 == -1)
                    dp[index][rest] = p1;
                else // p1 != -1 && p2 != -1
                    dp[index][rest] = min(p1, p2 + 1);
            }
        }
    }
    return dp[0][aim];
}
```
