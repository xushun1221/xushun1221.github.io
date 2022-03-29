# 【算法】图


## 图的定义
图大体可以由邻接表和邻接矩阵两种形式来表示。  
*可以用自己习惯的结构将图的算法全部实现一遍，当遇到图相关的问题，可以将数据描述转换为自己熟悉的结构，然后解题*  
这里用三个结构体来给出定义：  
```cpp
// 图 Graph
struct Graph {
	unordered_map<int, Node*> nodes; // 点集 序号-顶点
	unordered_set<Edge*> edges; // 边集
};

// 顶点 Node
struct Node {
	int value; // 数据
	int in;	   // 入度
	int out;   // 出度
	vector<Node*> nexts; // 邻接顶点
	vector<Edge*> edges; // 邻接边
	Node(int val) : value(val), in(0), out(0) {}
};

// 边 Edge
struct Edge {
	int weight; // 权重
	Node* from; // 起点
	Node* to;   // 终点
	Edge(int wei, Node* f, Node* t) : weight(wei), from(f), to(t) {}
};
```

-----

## 广度优先遍历
思路：  
1. 利用辅助队列
2. 从源节点开始依次按广度（宽度）入队，然后弹出，访问
3. 每弹出一个节点，把该节点所有没有入队的邻接点入队
4. 直到队列空

代码：  
```cpp
// 广度优先遍历 BFS 从node出发  
void bfs(Node* node) {  
	if (!node)  
		return;  
	queue<Node*> nqueue;  
	nqueue.push(node);  
	unordered_set<Node*> nset; //遍历过的节点  
	while (!nqueue.empty()) {  
		Node* cur = nqueue.front();  
		nqueue.pop();  
		cout << cur -> value << endl; // visit  
		for (auto n : cur -> nexts) {  
			if (!nset.count(n)) {  
				nset.insert(n);  
				nqueue.push(n);  
			}  
		}  
	}  
}
```

-----

## 深度优先遍历
思路：  
1. 使用辅助栈
2. 从源节点开始按深度依次将节点入栈，访问，然后弹出
3. 每弹出一个点，把该节点下一个没进过栈的邻接点入栈
4. 直到栈空

代码：  
```cpp
// 深度优先遍历 DFS 从node出发  
void dfs(Node* node) {  
	if (!node)  
		return;  
	stack<Node*> nstack;  
	nstack.push(node);  
	cout << node -> value << endl;  
	unordered_set<Node*> nset;  
	nset.insert(node);  
	while (!nstack.empty()) {  
		Node* cur = nstack.top();  
		nstack.pop();  
		for (auto n : cur -> nexts) {  
			if (!nset.count(cur)) {  
				nstack.push(cur);  
				nstack.push(n);  
				nset.insert(n);  
				cout << n -> value << endl;  
				break; // 不访问cur的其他邻接点  
			}  
		}  
	}  
}
```

-----

## 拓扑排序
思路：在有向图中找到一个入度为0的顶点，将其输出，擦除该顶点以及该顶点出发的所有边。再找下一个入度为0的点，直到点集为空。  
代码：  
```cpp
// 拓扑排序  
vector<Node*> sortedTopology(Graph& graph) {  
	unordered_map<Node*, int> inmap; // key: 顶点 value: 剩余的入度  
	queue<Node*> zeroqueue; // 入度为0的顶点入队  
	for (auto n : graph.nodes) { // n pair  
		inmap[n.second] = n.second -> in; // 登记所有顶点的入度  
		if (n.second -> in == 0)  
			zeroqueue.push(n.second);  
	}  
	vector<Node*> res; // 排序结果  
	while (!zeroqueue.empty()) {  
		Node* cur = zeroqueue.front();  
		zeroqueue.pop();  
		res.push_back(cur);  
		for (auto n : cur -> nexts) {  
			inmap[n] = inmap[n] - 1;  
			if (inmap[n] == 0)  
				zeroqueue.push(n);  
		}  
	}  
	return res;  
}
```

-----

## 最小生成树
**无向图**中的最小生成树：保持连通性，且边的权值总和最小。

### Kruskal算法
思路：以边的角度出发，给边排序，依次选择最小的边，如果该边加入边集不会让图成环，就加入，否则放弃。（如何判断加入边时是否会成环，需要判断边的两端顶点是否再一个“集合”中，这里需要用到并查集的结构） 
代码：（代码没有测试，可能有错）  
```cpp
// Kruskal  
// 简易并查集  
class mySet {  
public:  
	unordered_map<Node*, unordered_set<Node*>> setmap;  
	mySet(unordered_set<Node*> sm) {  
		for (auto n : sm) { // 开始每个顶点属于一个集合  
			unordered_set<Node*> s;  
			s.insert(n);  
			setmap[n] = s;  
		}  
	}  
	bool isSameSet(Node* from, Node* to) { // 两个顶点是否在一个集合  
		return setmap[from] == setmap[to];  
	}  
	void Union(Node* from, Node* to) { // 合并两个顶点所在集合  
		unordered_set<Node*> sf = setmap[from];  
		unordered_set<Node*> st = setmap[to];  
		for (auto n : st) {  
			sf.insert(n);  
			setmap[n] = sf;  
		}  
	}  
};  

// 比较器  
bool cmpEdge(const Edge* e1, const Edge* e2) {  
	return e1 -> weight > e2 -> weight;  
}  

unordered_set<Edge*> kruskalMST(Graph& graph) {  
	unordered_set<Node*> nodeset;  
	for (auto npair : graph.nodes)  
		nodeset.insert(npair.second);  
	mySet unionfind(nodeset);  
	priority_queue<Edge*, vector<Edge*>, decltype(&cmpEdge)> edgepriqueue;  // 小根堆
	for (auto e : graph.edges)  
		edgepriqueue.push(e);  
	unordered_set<Edge*> res;  
	while (!edgepriqueue.empty()) {  
		Edge* e = edgepriqueue.top();  
		edgepriqueue.pop();  
		if (!unionfind.isSameSet(e -> from, e -> to)) {  
			res.insert(e);  
			unionfind.Union(e -> from, e -> to);  
		}  
	}  
	return res;  // 返回边的集合
}
```

-----

### Prim算法
思路：以点的角度出发，初始时任意选一个顶点加入集合，加入集合的点邻接的边就被激活，从所有被激活的边中选择一条代价最小且它连接的点不在集合中，加入新的边后会将和它连接的顶点加入集合，新加入的顶点又会激活新的边，…，循环直到所有顶点被加入进来。  
代码：  
```cpp
// Prim  
unordered_set<Edge*> primMST(Graph& graph) {  
	// 解锁的边进入一个小根堆  
	priority_queue<Edge*, vector<Edge*>, decltype(&cmpEdge)> eminheap;  
	unordered_set<Node*> nset; // 点集  
	unordered_set<Edge*> res; // 结果 边集  
	for (auto n : graph.nodes) { // 随便挑一个点开始 n是pair  for是为了森林
		nset.insert(n.second); // 点加入点集  
		for (auto e : n.second -> edges) // 该点激活和它相连的边  
			eminheap.push(e);  
		while (!eminheap.empty()) {  
			Edge* e = eminheap.top(); // 代价最小的边  
			eminheap.pop();  
			Node* enode = e -> to; // 可能的新点  
			if (!nset.count(enode)) { // 新点不在点集中  
				nset.insert(enode);  
				res.insert(e);  
				for (auto e : enode -> edges) // 新点加入 激活新边  
					eminheap.push(e);  
			}  
		}  
	}  
	return res;  
}
```

-----

## 单源最短路径——Dijkstra算法
条件：没有权值为负的边
思路：维护一张源点到各顶点之间距离的表，初始情况集合中只有源点，每一轮添加一个最短的路径的终点进来，查看添加过后源点到其他顶点是否有更短的距离，如果存在更短距离，更新表项，然后确定添加的顶点距离不再改变。直到找到源点到终点的最短路径。  
代码：  
```cpp
// Dijkstra 经典写法  （还有更优写法 需要改写堆的实现）
// 在distmap中选一个最小距离的顶点 该顶点不能是被选过的  
Node* getMinDistAndUnselectedNote(unordered_map<Node*, int> distmap, unordered_set<Node*> selectedset) {  
	Node* minnode = nullptr;  
	int mindist = INT_MAX;  
	for (auto npair : distmap) {  
		Node* tn = npair.first;  
		int td = npair.second;  
		if (!selectedset.count(tn) && td < mindist) {  
			minnode = tn;  
			mindist = td;  
		}  
	}  
	return minnode;  
}  

unordered_map<Node*, int> dijkstra(Node* head) {  
	// head 源点 返回值是 距离表  
	unordered_map<Node*, int> distmap; // key: 顶点 value: 源点到该顶点最小距离  如果表中没有 则距离为正无穷  
	distmap[head] = 0; // 初始  
	unordered_set<Node*> selectedset; // 已经求过距离的节点不再动它  
	Node* minnode = getMinDistAndUnselectedNote(distmap, selectedset);  
	while (minnode) {  
		int dist = distmap[minnode];  
		for (auto e : minnode -> edges) { // 考查当前节点出去的边  
			Node* tonode = e -> to;  
			if (!distmap.count(tonode)) // 原来无穷远 更新  
				distmap[tonode] = dist + e -> weight;  
			distmap[e -> to] = min(distmap[tonode], dist + e -> weight); // 更新  
		}  
		selectedset.insert(minnode);  
		minnode = getMinDistAndUnselectedNote(distmap, selectedset);  
	}  
	return distmap;  
}
```

