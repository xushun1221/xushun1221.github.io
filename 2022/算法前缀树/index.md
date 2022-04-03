# 【算法】前缀树


## 前缀树的定义
前缀树（Trie，字典树）是**N叉树**的一种特殊形式。通常来说，一个前缀树是用来**存储字符串**的。前缀树的每一个节点代表一个**字符串（前缀）**。每一个节点会有多个子节点，通往不同子节点的路径上有着不同的字符。子节点代表的字符串是由节点本身的原始字符串，以及通往该子节点路径上所有的字符组成的。  
前缀树节点可以用数组或哈希表来存储子节点。这里给出一种实现：  
```cpp
class Trie {
private:
    int pass; // 经过该节点的字符串数量 该节点是多少字符串的前缀
    int end;  // 以该字符为结尾的字符串个数
    // 子节点数组 全部初始化为nullptr 0~25 : a~z 
    // 如果字符种类很多可以用哈希表 unordered_map<char, Trie*> nexts
    Trie* nexts[26] = {};
public:
    Trie() : pass(0), end(0) {}
    ~Trie() {
        for (auto node : nexts)
            if (node)
                delete node;
    }
    void insert(string word) {
        Trie* node = this;
        ++ node -> pass;
        int index = 0;
        for (auto ch : word) { // 走哪条路
            index = ch - 'a';
            if (!node -> nexts[index])
                node -> nexts[index] = new Trie();
            node = node -> nexts[index];
            ++ node -> pass;
        }
        ++ node -> end;
    }
    
    // 是否存在某个字符串 （某个字符串添加了几次）
    bool search(string word) {
        Trie* node = this;
        int index = 0;
        for (auto ch : word) {
            index = ch - 'a';
            if (!node -> nexts[index])
                return 0;
            node = node -> nexts[index];
        }
        return node -> end; // c++中非0整型到bool转换为true
    }
    
    // 是否存在某个字符串以prefix为前缀 （几个）
    bool startsWith(string prefix) {
        Trie* node = this;
        int index = 0;
        for (auto ch : prefix) {
            index = ch - 'a';
            if (!node -> nexts[index])
                return 0;
            node = node -> nexts[index];
        }
        return node -> pass;
    }
	
	// 从前缀树中删除一个字符串
	void deleteStr(string word) {
		if (search(word)) { // 删除字符串的前提是前缀树包含该串
			Trie* node = this;
			-- node -> pass;
			int index = 0;
			for (auto ch : word) { // 沿着路径删下去
				index = ch - 'a';
				if (-- node -> nexts[index] -> pass == 0) { // 这条路径上就只剩下要删除的串的剩下部分 就全部删除
					delete node -> nexts[index];
					node -> nexts[index] = nullptr;
					return;
				}
				node = node -> nexts[index];
			}
			-- node -> end; // 最后删掉串尾标识
		}
	}
};
```
这里可以测试[LeetCode208](https://leetcode-cn.com/problems/implement-trie-prefix-tree/)  

-----
