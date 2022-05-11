# 【算法】二叉树


## 二叉树定义
```cpp
struct TreeNode {
	int val;
	TreeNode *left;
	TreeNode *right;
	// TreeNode() : val(0), left(nullptr), right(nullptr) {}
	// TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
	// TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
};
```

-----

## 递归遍历
```cpp
// 先序
void PreOrder(TreeNode *root) {
	if (!root)
		return;
	cout << root -> val;
	PreOrder(root -> left);
	PreOrder(root -> right);
}

// 中序
void InOrder(TreeNode *root) {
	if (!root)
		return;
	InOrder(root -> left);
	cout << root -> val;
	InOrder(root -> right);
}

// 后序
void PostOrder(TreeNode *root) {
	if (!root)
		return;
	PostOrder(root -> left);
	PostOrder(root -> right);
	cout << root -> val;
}
```

-----

## 非递归遍历
### 先序
思路：使用一个辅助栈  
1. 从栈中弹出一个节点
2. 访问这个 节点
3. 右、左孩子入栈（如果有）
4. 回到 1.

代码：  
```cpp
void PreOrder(TreeNode *root) {
	if (root) {
		stack<TreeNode*> tnstack;
		tnstack.push(root);
		while (!tnstack.empty()) {
			TreeNode *t = tnstack.top();
			tnstack.pop();
			cout << t -> val;
			if (t -> right)
				tnstack.push(t -> right);
			if (t -> left)
				tnstack.push(t -> left);
		}
	}
}
```

-----

### 后序
思路：由先序遍历拓展，先序遍历出栈顺序为（根-左-右），将其调整为（根-右-左），如果我们出栈时，不访问，而是再将节点按（根-右-左）的顺序放入另一个栈中，那么这个栈的出栈顺序就是（左-右-根），就是后序遍历序列。  
使用两个栈  
1. 从栈1中弹出一个节点（不访问）
2. 将其压入栈2
3. 同时将它的左、右孩子分别入栈1
4. 回到 1.

代码：  
```cpp
void PostOrder(TreeNode *root) {
	if (root) {
		stack<TreeNode*> s1;
		stack<TreeNode*> s2;
		s1.push(root);
		while (!s1.empty()) {
			TreeNode *t = s1.top();
			s1.pop();
			s2.push(t);
			if (t -> left)
				s1.push(t -> left);
			if (t -> right)
				s1.push(t -> right);
		}
		while (!s2.empty()) {
			cout << s2.top() -> val;
			s2.pop();
		}
	}
}
```

-----

### 中序
思路：使用一个辅助栈  
1. 整棵树的左边界进栈
2. 从栈中弹出一个节点，兵访问
3. 如果该节点有右孩子，右孩子进栈
4. 回到 1.

代码：  
```cpp
void InOrder(TreeNode *root) {
	if (root) {
		stack<TreeNode*> tnstack;
		while (!tnstack.empty() || root) {
			if (root) { // 左边界进栈
				tnstack.push(root);
				root = root -> left;
			}
			else { // 向左到头了 打印 并指向右孩子
				root = tnstack.top();
				tnstack.pop();
				cout << root -> val;
				root = root -> right;
			}
		}
	}
}
```

-----

## 层序遍历（广度优先遍历）
思路：使用一个辅助队列  
1. 从队列中弹出一个节点
2. 访问该节点
3. 节点的左、右孩子入队
4. 回到 1.

代码：  
```cpp
void LevelOrder(TreeNode *root) {
	if (root) {
		queue<TreeNode*> tnqueue;
		tnqueue.push(root);
		while (!tnqueue.empty()) {
			TreeNode *t = tnqueue.front();
			tnqueue.pop();
			cout << t -> val;
			if (t -> left)
				tnqueue.push(t -> left);
			if (t -> right)
				tnqueue.push(t -> right);
		}
	}
}
```

-----

### 题目：求一颗二叉树的最大宽度
题目描述：求一颗二叉树的最大宽度。  
思路：按层序遍历的方法遍历，使用一个哈希表，记录节点和对应的层数，每遍历一层便统计该层的个数，获得最大值即可。

代码：
```cpp
int widthOfBTree(TreeNode *root) {
	if (!root)
		return 0;
	queue<TreeNode*> tnqueue;
	unordered_map<TreeNode*, int> levelmap;
	tnqueue.push(root);
	levelmap[root] = 1;
	int curlevel = 1;
	int curlevelnodes = 0;
	int maxwidth = INT_MIN;
	while (!tnqueue.empty()) {
		TreeNode *curnode = tnqueue.front();
		int curnodelevel = levelmap[curnode];
		if (curnodelevel == curlevel)
			++ curlevelnodes;
		else {
			maxwidth = max(maxwidth, curlevelnodes);
			++ curlevel;
			curlevelnodes = 1;
		}
		if (curnode -> left) {
			levelmap[curnode -> left] = curnodelevel + 1;
			tnqueue.push(curnode -> left);
		}
		if (curnode -> right) {
			levelmap[curnode -> right] = curnodelevel + 1;
			tnqueue.push(curnode -> right);
		}
	}
	return max(maxwidth, curlevelnodes);
}
```

-----

## 相关题目
### 递归方法求解二叉树问题的套路
- 递归调用时需要左、右子树分别提供什么信息
- 如果对左右子树提供信息的要求不同，看是否能统一要求（因为用的是递归方法）
- 通常我们可以定义一个返回数据类来包含需要返回的信息
- 这种方法可以解决所有树状DP问题（左右子树提供的信息就是DP）

-----

### 题目1：判断一颗二叉树是否为搜索二叉树
[LeetCode98](https://leetcode-cn.com/problems/validate-binary-search-tree/submissions/)  
思路：按中序遍历的序列来考虑，当前访问的节点比上次访问的节点大即可，否则返回false。  
代码：  
```cpp
// 非递归方法
bool isValidBST(TreeNode* root) {
	if (root) {
		stack<TreeNode*> tnstack;
		long preValue = LONG_MIN;
		while (root || !tnstack.empty()) {
			if (root) {
				tnstack.push(root);
				root = root -> left;
			}
			else {
				TreeNode * t = tnstack.top();
				tnstack.pop();
				if (t -> val <= preValue)
					return false;
				else
					preValue = t -> val;
				root = t -> right;
			}
		}
	}
	return true;
}

// 递归方法
long preValue = LONG_MIN; 
bool isValidBST(TreeNode* root) {
	if (!root)
		return true;
	bool leftBST = isValidBST(root -> left);
	if (!leftBST)
		return false;
	if (root -> val <= preValue)
		return false;
	else
		preValue = root -> val;
	return isValidBST(root -> right);
}
```

思路2：递归套路，我们需要知道：  
1. 左、右子树是否是搜索二叉树
2. 左子树的最大值，小于，当前节点
3. 右子树的最小值，大于，当前节点

可以看到，对于左右子树的要求信息不相同，那么我们统一要求，对左右子树都要求这些：  
1. 是否搜索二叉树
2. 子树的最大值
3. 子树的最小值

代码：  
```cpp
class ReturnData {
	public:
		bool isBST;
		int maxVal;
		int minVal;
		ReturnData(bool bst, int maxv, int minv) : isBST(bst), maxVal(maxv), minVal(minv) {}
};

ReturnData process(TreeNode * root) {
	if (!root) // base case
		return ReturnData(true, 0, 1); // 树空时 用maxVal<minVal来标记
	// 抓取左右子树的信息
	ReturnData leftData = process(root -> left);
	ReturnData rightData = process(root -> right);
	// 处理当前树的信息
	int maxv = root -> val;
	int minv = root -> val;
	if (leftData.maxVal >= leftData.minVal) { // 左子树不为空
		maxv = max(leftData.maxVal, maxv);
		minv = min(leftData.minVal, minv);
	}
	if (rightData.maxVal >= rightData.minVal) { // 右子树不为空
		maxv = max(rightData.maxVal, maxv);
		minv = min(rightData.minVal, minv);
	}
	bool isBST = true;
	if (leftData.maxVal >= leftData.minVal && (!leftData.isBST || leftData.maxVal >= root -> val))
		isBST = false;
	if (rightData.maxVal >= rightData.minVal && (!rightData.isBST || rightData.minVal <= root -> val))
		isBST = false;
	// 返回
	return ReturnData(isBST, maxv, minv);
}

bool isValidBST(TreeNode* root) {
	return process(root).isBST;
}
```

-----

### 题目2：判断一颗二叉树是否为完全二叉树
[LeetCode958](https://leetcode-cn.com/problems/check-completeness-of-a-binary-tree/)  
思路：按层序遍历的顺序遍历，使用辅助队列，分为三种情况：  
1. 左右孩子都有，跳过；
2. 只有右孩子，没有左孩子，返回false；
3. 不满足2.的情况下，如左右孩子不全，那么自该节点开始，后续的节点都必须为叶子节点。

代码：  
```cpp
bool isCompleteTree(TreeNode* root) {
	if (!root)
		return true;
	queue<TreeNode*> tnqueue;
	tnqueue.push(root);
	bool leaf = false; // 是否遇到左右不全的节点
	while (!tnqueue.empty()) {
		TreeNode *t = tnqueue.front();
		tnqueue.pop();
		if ( (t -> right && !t -> left) || (leaf && (t -> left || t -> right)))
			return false;
		if (t -> left)
			tnqueue.push(t -> left);
		if (t -> right)
			tnqueue.push(t -> right);
		if (!t -> left || !t -> right)
			leaf = true;
	}
	return true;
}
```

-----

### 题目3：判断一颗二叉树是否为满二叉树
思路：可以用节点个数`n`和二叉树深度`h`来判断，如果符合$n=2^{h}-1$那么就是满二叉树。我们可以用我们的递归套路来解这道题。我们对于左右子树都需要：树的高度、树的结点个数。  
代码：  
```cpp
class ReturnData {
	public:
		int nodes;
		int height;
		ReturnData(int n, int h) : nodes(n), height(h) {}
};

ReturnData process(TreeNode * root) {
	if (!root) // base case
		return ReturnData(0, 0);
	// 抓取左右子树的信息
	ReturnData leftData = process(root -> left);
	ReturnData rightData = process(root -> right);
	// 处理当前树的信息
	int nodes = leftData.nodes + rightData.nodes + 1;
	int height = max(leftData.height, rightData.height) + 1;
	// 返回
	return ReturnData(nodes, height);
}

bool isFull(TreeNode* root) {
	if (!root)
		return true;
	ReturnData t = process(root);
	return t.nodes == (1 << t.height) - 1;
}
```

-----

### 题目4：判断一颗二叉树是否为平衡二叉树
[LeetCode110](https://leetcode-cn.com/problems/balanced-binary-tree/)  
思路：递归套路，知道左右子树是否为平衡二叉树以及他们的高度即可。  
代码：  
```cpp
class ReturnData {
	public:
		bool isBal;
		int height;
		ReturnData(bool b, int h) : isBal(b), height(h) {}
};

ReturnData process(TreeNode * root) {
	if (!root)
		return ReturnData(true, 0);
	ReturnData leftData = process(root -> left);
	ReturnData rightData = process(root -> right);
	int height = max(leftData.height, rightData.height) + 1;
	bool isBal = leftData.isBal && rightData.isBal && abs(leftData.height - rightData.height) <= 1;
	return ReturnData(isBal, height);
}

bool isBalanced(TreeNode* root) {
	return process(root).isBal;
}
```

-----

### 题目5：最低公共祖先节点
[LeetCode236](https://leetcode-cn.com/problems/lowest-common-ancestor-of-a-binary-tree/)  
题目描述：给定两个二叉树的节点`p  q`，求它们的最低公共祖先节点。  
思路1：用一个哈希表来记录节点的父节点，用一个哈希set来从下往上记录`p`节点的路径上的节点，然后从下往上检查`q`节点路径上的节点在不在这个集合里，第一个找到的就是最低公共祖先。  
代码：  
```cpp
unordered_map<TreeNode*, TreeNode*> father;
unordered_set<TreeNode*> ppath;
void process(TreeNode * root) { // get father
	if (!root)
		return;
	father[root -> left] = root;
	father[root -> right] = root;
	process(root -> left);
	process(root -> right);
}
TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
	father[root] = nullptr;
	process(root);
	while (p) {
		ppath.insert(p);
		p = father[p];
	}
	while (q) {
		if (ppath.count(q))
			return q;
		q = father[q];
	}
	return nullptr;
}
```

思路2：第一种方法比较直观，但需要记录所有的父节点，我们还可以用另一种递归的方式来解决问题。考虑这两种情况：  
1. `q`是`p`的最低公共祖先（反过来也一样），也就是从`p`到`root`的路径上经过`q`
2. `q`和`p`不互为最低公共祖先，也就是来自两颗子树，汇聚到一起

算法从下往上递归，遇到`q`或`p`就沿着路径向上，如果汇聚到一起，就返回该节点；如果一个是另一个的祖先，当访问祖先时，就会将祖先向上返回到递归结束。  
代码：  
```cpp
TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
	if (!root || root == p || root == q)
		return root;
	TreeNode * left = lowestCommonAncestor(root -> left, p, q);
	TreeNode * right = lowestCommonAncestor(root -> right, p, q);
	if (left && right)
		return root;
	return left ? left : right;
}
```

-----

### 问题6：求一个二叉树节点的中序后继节点
问题描述：给定一个二叉树中的节点，返回该节点的后续节点，后续节点定义为中序序列中的下一个节点。节点类型如下：(`parent`指针指向父节点，根节点的`parent`指针为空)  
```cpp
struct TreeNode {
	int val;
	TreeNode *left;
	TreeNode *right;
	TreeNode *parent;
};
```
思路：中序遍历的顺序为，左根右，考虑两种情况：  
1. 当前节点存在右子树，那么下一个节点应为它右子树中第一个中序节点（也就是右子树中最左的那个节点）
2. 当前节点没有右子树，那么以当前节点为根的二叉树已经遍历完，下一个遍历的节点应该沿着`parent`指针向上寻找，直到找到某个节点`p`成为它`parent`的左孩子，那么意味着，`p`为根的子树已经遍历完，下一个遍历的节点即为`p`的`parent`，也就是当前节点的后继
3. 上一种情况的特例，如果当前节点向上，没有找到一个节点成为某个节点的左孩子，那么证明当前节点为整颗树的最右下节点，则没有后继节点。

代码：  
```cpp
TreeNode* getLeftMost(TreeNode* node) { // 找到最左的节点
	if (!node)
		return node;
	while (node -> left)
		node = node -> left;
	return node;
}

TreeNode* getSuccessorNode(TreeNode* node){
	if (!node)
		return node;
	if (node -> right) { // 存在右子树
		return getLeftMost(node -> right);
	}
	else {	// 不存在右子树
		TreeNode* parent = node -> parent;
		while (parent && parent -> left != node) {
			node = parent;
			parent = parent -> parent;
		}
		return parent;
	}
	return nullptr;
}
```

-----

### 问题7：二叉树的序列化和反序列化
[LeetCode297](https://leetcode-cn.com/problems/serialize-and-deserialize-binary-tree/)  
问题描述：二叉树的序列化是将在内存中的二叉树转化为字符串的形式，而反序列化时将字符串还原为内存中的二叉树。  
思路：使用先序遍历（其他遍历方式也可）方法，用`#`表示空节点，`_`下划线跟在节点后表示节点的结束。反序列化时使用一个队列来保存序列中的节点信息，每次建立一个节点就弹出一个。  
代码：  
```cpp
string serialize(TreeNode* root) {
	string res;
	if (!root)
		return "#_";
	res = to_string(root -> val) + "_";
	res += serialize(root -> left);
	res += serialize(root -> right);
	return res;
}

TreeNode* process(queue<string>& dataqueue) {
	string t = dataqueue.front();
	dataqueue.pop();
	if (t == "#")
		return nullptr;
	TreeNode* node = new TreeNode(stoi(t));
	node -> left = process(dataqueue);
	node -> right = process(dataqueue);
	return node;
}

TreeNode* deserialize(string data) {
	char* str = new char[data.size() + 1];
	strcpy(str, data.c_str());
	char* d = "_";
	char* p = strtok(str, d);
	queue<string> dataqueue;
	while (p) {
		string s = p;
		dataqueue.push(s);
		p = strtok(nullptr, d);
	}
	return process(dataqueue);
}
// 自己写的
// TreeNode* deserialize(string data) {
// 	queue<string> dataQueue;
// 	string cur = "";
// 	for (auto ch : data) {
// 		if (ch != '_') {
// 			cur += ch;
// 		} else if (cur != "") { // ch == '_'
// 			dataQueue.push(cur);
// 			cur = "";
// 		}
// 	}
// 	return process(dataQueue);
// }
```

-----

### 问题8：折纸问题
问题描述：把一段纸条竖着放在桌子上，然后从纸条的下边向上边对折1次，压出折痕后展开。此时折痕是凹下去的，即折痕突起的方向指向纸条的背面。如果从纸条的下边向上边连续对折2次，压出折痕后展开，此时有三条折痕，从上到下依次是，凹折痕、凹折痕、凸折痕。给定一个参数`n`，代表纸条向上对折的次数，请从上到下打印所有折痕的方向。例如：`n == 1`，打印，`down`；`n == 2`，打印，`down donw up`。  
思路：观察每次的折痕，可以看出，从第二次开始，每次对折，会在上一次对折所生成的折痕（每个）的上、下，分别生成一个凹、凸折痕，如，第一次折痕的上方是一个凹，下方是一个凸。可以用二叉树来表示折痕的情况，每一层节点就是每一次的折痕，这是一颗这样的树：总根是凹，每一个左子树的根为凹，每一个右子树的根为凸。从上到下打印，其实就相当于二叉树的中序遍历。  
代码：  
```cpp
void process(int level, int n, bool down) {
	if (i > n)
		return;
	process(i + 1, n, true); // 折上面凹的那个
	cout << down ? "down" : "up" << endl; // 打印当前
	process(i + 1, n, false); // 折下面凸的那个
}

void fold(int n) {
	process(1, n, true);
}
```

-----
