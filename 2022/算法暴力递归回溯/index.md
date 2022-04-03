# 【算法】暴力递归（回溯）


## 暴力递归
暴力递归就是尝试：  
1. 把问题转化为规模缩小了的同类问题的子问题
2. 有明确的不需要继续进行递归的条件（base case）
3. 有当得到了子问题的结果之后的决策过程
4. 不记录每个子问题的解

*本章记录的题目都使用暴力递归方式解题，可能存在动态规划等更好的方法。*

-----

## 经典问题：N皇后
[LeetCode52](https://leetcode-cn.com/problems/n-queens-ii/)  
问题描述：n 皇后问题 研究的是如何将 `n` 个皇后放置在 `n×n` 的棋盘上，并且使皇后彼此之间不能相互攻击（皇后之间不能共行、共列、共斜线）。给你一个整数 `n` ，返回所有不同的 `n` 皇后问题 的解决方案的数量。  
思路：暴力递归，从第`0`行开始到第`n-1`行，依次确定皇后放置位置剩下的皇后有多少可能情况。  
代码：  
```cpp
bool isValid(int i, int j, int n, vector<int>& record) {
	for (int k = 0; k < i; ++ k)
		if (j == record[k] || abs(record[k] - j) == abs(k - i))
			return false;
	return true;
}
int process(int i, int n, vector<int>& record) {
	if (i == n)
		return 1;
	int res = 0;
	for (int j = 0; j < n; ++ j) {
		if (isValid(i, j, n, record)) {
			record[i] = j;
			res += process(i + 1, n, record);
		}
	}
	return res;
}
int totalNQueens(int n) {
	vector<int> record(n, -1);
	return process(0, n, record);
}
```
时间复杂度：$O\left(n^{2}\right)$  
空间复杂度：$O\left(n\right)$  

基于位运算加速的优化方法：用位运算来确定可以尝试的位置。确定每一行的皇后位置时，可以知道下一行皇后不可以出现的位置（优化了常数项的时间）  
代码：  
```cpp
int process(int Lim, int colLim, int leftDiaLim, int rightDiaLim) {
	// Lim 寻找空间限制
	// colLim 列方向限制
	// leftDiaLim 左斜线方向限制
	// rightDiaLim 右斜线方向限制
	// n个皇后全部找到位置 返回一种情况
	if (colLim == Lim)
		return 1;
	int mostRightOne = 0;
	// 可以放皇后的位置为1
	int pos = Lim & (~(colLim | leftDiaLim | rightDiaLim));
	int res = 0;
	while (pos != 0) {
		// 选择可选位置的最右位置
		mostRightOne = pos == INT_MIN ? pos : pos & (~pos + 1);
		// 删除这个位
		pos = pos - mostRightOne;
		res += process(Lim,
						colLim | mostRightOne,              // 列限制 
						(leftDiaLim | mostRightOne) << 1,   // 左斜线 
						(rightDiaLim | mostRightOne) >> 1); // 右斜线 
	}
	return res;
}
int totalNQueens(int n) {
	// 使用位运算来限制寻找空间 皇后数量不能超过整型位数
	if (n < 1 || n > 32)
		return 0;
	// 32位整型的最后n位为1
	int Lim = n == 32 ? -1 : (1 << n) - 1;
	return process(Lim, 0, 0, 0);
}
```
时间复杂度：$O\left(n^{2}\right)$  
空间复杂度：$O\left(n\right)$  

-----

## 相关题目
### 题目1：打印一个子串的全部子序列 包括空串 
思路：其实就是求子集，从字符串开头到结尾，依次选择要或不要。穷尽所有可能。  
代码：  
```cpp
void process(string str, int index, vector<char>& res) {
	if (index == str.size()) {
		for (auto ch : res)
			cout << ch;
		return;
	}
	vector<char> resKeep(res.begin(), res.end());
	resKeep.push_back(str[index]);
	process(str, index + 1, resKeep);
	vector<char> resNoKeep(res.begin(), res.end());
	process(str, index + 1, resNoKeep);
}

void allSubseq(string str) {
	vector<char> res;
	process(str, 0, res);
}

// 优化的算法 省空间
void process(string str, int index) {
	if (index == str.size()) {
		for (auto ch : res)
			if (ch != 0)
				cout << ch;
		return;
	}
	process(str, index + 1);
	char tmp = str[index];
	str[index] = 0;
	process(str, index + 1);
	str[index] = tmp;
}
```

-----

### 题目2：字符串的全排列
[LeetCode-剑指Offer38](https://leetcode-cn.com/problems/zi-fu-chuan-de-pai-lie-lcof/)  
题目描述：输入一个字符串，打印出该字符串中字符的所有排列，不能有重复字符串。  
思路：在字符串中可以选择每一个字符当头，剩下的字符继续全排列。如何实现不重复？可以使用哈希表，但是有点笨。思考一下为什么会重复，原因是每次循环选择头字符的时候，可能会有一样的，那我们在循环的时候记录一下选择过的头字符，再次遇到直接跳过即可。  
代码：  
```cpp
void process(string s, int index, vector<string>& res) {
	if (index == s.size()) {
		res.push_back(s);
		return;
	}
	vector<bool> count(26, false);
	for (int i = index; i < s.size(); ++ i) {
		if (!count[s[i] - 'a']) {
			swap(s[i], s[index]);
			process(s, index + 1, res);
			swap(s[i], s[index]);
			count[s[i] - 'a'] = true;
		}
	}
}
vector<string> permutation(string s) {
	vector<string> res;
	process(s, 0, res);
	return res;
}
```

-----

### 题目3：纸牌博弈-预测赢家
[LeetCode486](https://leetcode-cn.com/problems/predict-the-winner/)  
题目描述：给你一个整数数组 `nums` 。玩家 1 和玩家 2 基于这个数组设计了一个游戏。玩家 1 和玩家 2 轮流进行自己的回合，玩家 1 先手。开始时，两个玩家的初始分值都是 `0` 。每一回合，玩家从数组的任意一端取一个数字（即，`nums[0]` 或 `nums[nums.length - 1]`），取到的数字将会从数组中移除（数组长度减 1 ）。玩家选中的数字将会加到他的得分上。当数组中没有剩余数字可取时，游戏结束。如果玩家 1 能成为赢家，返回 `true` 。如果两个玩家得分相等，同样认为玩家 1 是游戏的赢家，也返回 `true` 。你可以假设每个玩家的玩法都会使他的分数最大化。  
思路：递归，先手情况下选择一个数，然后转为后手情况；后手情况下，对方先选一个数，然后转为先手情况。  
代码：  
```cpp
int first(vector<int>& nums, int l, int r) { // 先手
	if (l == r) // base case
		return nums[l];
	// 先手可以选择拿左边或右边的牌 绝对聪明的情况下 会拿结果最大的
	return max(nums[l] + second(nums, l + 1, r), nums[r] + second(nums, l, r - 1));
}
int second(vector<int>& nums, int l, int r) { // 后手
	if (l == r) // base case
		return 0;
	// 后手要在先手选择完后再选择 先后选择左或右
	// 剩下两种范围 后手转为先手选择 因为后手是后选
	// 所以先手会留下后手范围小的那个
	return min(first(nums, l + 1, r), first(nums, l, r - 1));
}
bool PredictTheWinner(vector<int>& nums) {
	return first(nums, 0, nums.size() - 1) >= second(nums, 0, nums.size() - 1);
}
```
如何改为动态规划再说。

-----

### 题目4：逆序一个栈
题目描述：逆序一个栈，不能用额外的结构，只能使用递归函数。  
思路：用一个递归函数完成将栈底元素拿出来的操作（其他元素位置不变），改函数接收栈顶弹出的元素，这样保存了栈顶元素，然后进入下一层递归，也这样操作，当栈空时，返回保存的栈顶元素，当函数接收到下层递归函数传来的元素，将之前保存的栈顶元素压栈，然后返回上层返回的元素，最后就可以返回栈底元素了。然后再用一个递归函数调用上面的方法，获取栈底元素，然后再递归调用该函数，当栈空时，返回上层，上层发现下层结束后，将保存的元素压栈，即可。  
代码：  
```cpp
void reverse(stack<int>& s) {
	if (s.empty())
		return;
	int end = f(s);
	reverse(s);
	stack.push(end);
}
int f(stack<int>& s) {
	int res = s.top();
	s.pop();
	if (s.empty())
		return res;
	int last = f(s);
	s.push(res);
	return last;
}
void reStack(stack<int>& s) {
	reverse(s);
}
```

-----

### 题目5：把数字翻译为字符串
[LeetCode-剑指Offer46](https://leetcode-cn.com/problems/ba-shu-zi-fan-yi-cheng-zi-fu-chuan-lcof/)  
题目描述：给定一个数字，我们按照如下规则把它翻译为字符串：0 翻译成 “a” ，1 翻译成 “b”，……，11 翻译成 “l”，……，25 翻译成 “z”。一个数字可能有多个翻译。请编程实现一个函数，用来计算一个数字有多少种不同的翻译方法。例如，`12258`，可以翻译为5种字符串，`"bccfi", "bwfi", "bczi", "mcfi", "mzi"`。  
思路：还是递归，从左往右。每次遍历只考虑从当前下标到数组最后范围有多少种可能。  
代码：  
```cpp
int process(vector<int>& nums, int index) {
	if (index == nums.size())
		return 1;
	if (nums[index] == 0)
		return process(nums, index + 1);
	if (nums[index] >= 3)
		return process(nums, index + 1);
	int res = 0;
	if (nums[index] == 1 || nums[index] == 2) {
		if (index == nums.size() - 1)
			res = process(nums, index + 1);
		else if (10 * nums[index] + nums[index + 1] > 25)
			res = process(nums, index + 1);
		else 
			res = process(nums, index + 1) + process(nums, index + 2);
	}
	return res;
}
int translateNum(int num) {
	vector<int> nums;
	while (num) {
		nums.insert(nums.begin(), num % 10); // vector不支持头插 使用insert 在迭代器之前插入
		num /= 10;
	}
	return process(nums, 0);
}
```

-----

### 题目6：01背包问题
问题描述：给定两个数组分别存放重量和价值：`weights[] values[]`，下标对应，给定一个最大重量限制`bag`，可以选择每个货物是否装入背包（不可分割），求最大价值。  
思路：递归，从左往右，要还是不要。  
代码：  
```cpp
int process(vector<int>& weights, vector<int>& values, int index, int weightAlready, int valueAlraedy, int bag) {
	if (weightAlready > bag) // 这条路要放弃
		return 0;
	if (index = weights.size())
		return valueAlready;
	if (weights[i] + weightAlready <= bag) // 剩余容量可以要这个货
		return max(process(weights, values, index + 1, weightAlready + weights[index], valueAlready + values[index], bag), process(weights, values, index + 1, weightAlready, valueAlready, bag));
	else
		return process(weights, values, index + 1, weightAlready, bag);
}
int maxValue(vector<int>& weights, vector<int>& values, int bag) {
	return process(weights, values, 0, 0, 0, bag);
}
```

-----

