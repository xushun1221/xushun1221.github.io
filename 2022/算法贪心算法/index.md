# 【算法】贪心算法


## 贪心算法定义
在某个标准下，优先考虑最满足标准的样本，最后考虑最不满足标准的样本，最终得到一个答案的做法叫贪心算法。也就是说，不从整体最优上考虑，所做出的是在某种意义上的局部最优解。如果要得到全局最优解，需要局部最优到整体最优的证明。

## 经典题目：会议室
[LeetCode252](https://leetcode-cn.com/problems/meeting-rooms/submissions/)  
题目描述：一些项目要占用一个会议室宣讲，会议室不能同时容纳两个项目的宣讲。给你每一个项目的开始时间和结束时间，你来安排宣讲的日程，要求会议室进行的宣讲的场次最多，返回这个最多的场次。  
思路：按宣讲的结束时间从前往后排序，每次安排一个宣讲结束时间最靠前的，同时放弃那些时间冲突的宣讲。  
代码：  
```cpp
static bool cmp(vector<int>& a, vector<int>& b) {
	return a[1] < b[1];
}

int bestArrange(vector<vector<int>>& programs, int timePoint) {
	// program 宣讲安排 timepoint 开始时间
	sort(programs.begin(), programs.end(), cmp);
	int res = 0;
	for (int i = 0; i < programs.size(); ++ i) {
		if (timePoint <= programs[i][0]) {
			++ res;
			timePoint = program[i][1];
		}
	}
	return res;
}
```

-----

## 贪心算法笔试解题套路
1. 实现一个不依靠贪心的算法，可以暴力
2. 想出一个贪心算法
3. 用暴力算法和对数器去验证贪心算法是否正确
4. 不要纠结于贪心策略的正确性证明

-----
## 相关题目
### 题目1： 拼接所有字符串产生字典序最小的字符串
[NC85](https://www.nowcoder.com/practice/f1f6a1a1b6f6409b944f869dc8fd3381?tpId=196&tqId=37148&rp=1&ru=/exam/oj&qru=/exam/oj&sourceUrl=%2Fexam%2Foj%3Ftab%3D%25E7%25AE%2597%25E6%25B3%2595%25E7%25AF%2587%26topicId%3D196%26page%3D1%26search%3D%25E5%25AD%2597%25E5%2585%25B8%25E5%25BA%258F&difficulty=undefined&judgeStatus=undefined&tags=&title=%E5%AD%97%E5%85%B8%E5%BA%8F)  
题目描述：给定一个长度为 `n` 的字符串数组 `strs` ，请找到一种拼接顺序，使得数组中所有的字符串拼接起来组成的字符串是所有拼接方案中字典序最小的，并返回这个拼接后的字符串。  
思路：使用贪心策略，将所有字符串按某种比较策略排序，然后再拼接起来，比较策略为：将两个字符串拼接起来（前后两种顺序拼接），字典序小的在前。（该贪心策略的正确性在于排序方法具有传递性）  
代码：  
```cpp
static bool cmp(string a, string b) {
	return (a + b).compare(b + a) < 0;
}
string minString(vector<string>& strs) {
	string res;
	sort(strs.begin(), strs.end(), cmp);
	for (auto s : strs)
		res += s;
	return res;
}
```

-----

### 题目2：切割金条的最小成本
和这个题差不多：[LeetCode1547](https://leetcode-cn.com/problems/minimum-cost-to-cut-a-stick/)  
题目描述：给一个金条，按照要求切割成长度不一的几段，每次切割需要支付铜板，切割的代价就是当前要切割的金条的长度，求最小的切割成本。  
思路：可以反向思考，从所有切好的金条中组合两个最小的金条，成本为组合后的长度，然后，将继续组合直到变成一根金条即可。（利用了哈夫曼编码树的思想）  
代码：  
```cpp
int minCost(vector<int>& lens) {
	priority_queue<int, vector<int>, greater<int>> pq;
	for (auto i : lens)
		pq.push(i);
	int sum = 0, cur = 0;
	while (pq.size() > 1) {
		cur = pq.top(); pq.pop();
		cur += pq.top(); pq.pop();
		sum += cur;
		pq.push(cur);
	}
	return sum;
}
```

-----
