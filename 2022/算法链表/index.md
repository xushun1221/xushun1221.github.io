# 【算法】链表


## 链表定义
```cpp
// 单链表
struct ListNode {
	int val;
	ListNode *next;
};

// 双链表
struct ListNode {
	int val;
	ListNode *next;
	ListNode *front;
};
```

-----

## 链表题目之方法论
- 笔试：不用太在乎空间复杂度，一切为了时间复杂度
- 面试：时间复杂度依然放在第一位，尽量找到空间最省的方法
- 使用额外的数据结构来记录数据（哈希表等）
- 快慢指针

-----

## 相关题目
### 题目1：判断一个单链表是否为回文链表
[LeetCode234](https://leetcode-cn.com/problems/palindrome-linked-list/)  
题目描述：给你一个单链表的头节点 `head` ，请你判断该链表是否为回文链表。如果是，返回 `true` ；否则，返回 `false` 。
基本要求：时间复杂度$O\left(n\right)$  空间复杂度$O\left(n\right)$  
进阶要求：时间复杂度$O\left(n\right)$  空间复杂度$O\left(1\right)$  

基本思路：使用快慢指针找到链表中点，中点向后的节点入栈，出站时和链表前半部分节点比较即可  
基本代码：  
```cpp
 bool isPalindrome(ListNode* head) {
	ListNode * fast = head, * slow = head;
	while (fast -> next && fast -> next -> next) { // 快慢指针 
		fast = fast -> next -> next;			   // 慢指针 指向中点（奇数） 中点的前面（偶数）
		slow = slow -> next;
	}
	ListNode * p = slow -> next;
	stack<ListNode*> nodestack;  // 入栈
	while (p) {
		nodestack.push(p);
		p = p -> next;
	}
	while (!nodestack.empty()) { // 出栈比较
		ListNode * t = nodestack.top();
		nodestack.pop();
		if (head -> val != t -> val)
			return false;
		head = head -> next;
	}
	return true;
 }
```
时间复杂度$O\left(n\right)$   
空间复杂度$O\left(n\right)$  

进阶思路：快慢指针找到链表中点后，将链表的后半部分逆序（例如：1 -> 2 -> 3 -> 2 -> 1  变为  1 -> 2 -> 3 <- 2 <- 1），然后两个指针分别从前后向中间走，发现不一样即为非回文链表，最后将链表还原到初始情况  
进阶代码：  
```cpp
 bool isPalindrome(ListNode* head) {
	ListNode * n1 = head, * n2 = head;
	while (n2 -> next && n2 -> next -> next) {
		n1 = n1 -> next; // n1 -> mid
		n2 = n2 -> next -> next;// n2 -> end
	}
	n2 = n1 -> next; // right part
	n1 -> next = nullptr; // mid -> next == nullptr
	ListNode * n3 = nullptr;
	while (n2) {
		n3 = n2 -> next; // save next node
		n2 -> next = n1;
		n1 = n2;
		n2 = n3;
	}
	n3 = n1; // save last node    n1 -> right last node
	n2 = head; // n2 -> left first node
	bool res = true;
	while (n1 && n2) {
		if (n2 -> val != n1 -> val) {
			res = false;
			break;
		}
		n2 = n2 -> next; // left to mid
		n1 = n1 -> next; // right to mid
	}
	// restore to original linklist
	// n1 = n3 -> next;
	// n3 -> next = nullptr;
	// while (n1) {
	//     n2 = n1 -> next;
	//     n1 -> next = n3;
	//     n3 = n1;
	//     n1 = n2;
	// }
	return res;
 }
```
时间复杂度$O\left(n\right)$   
空间复杂度$O\left(1\right)$  

-----

### 问题2：将单链表按某个值划分成左边小、中间相等、右边大的形式
问题描述：给定一个单链表的头节点`head`，节点值为整型，再给定一个`pivot`，实现一个调整链表的算法，使得链表的左部分小于`pivot`，中间部分等于`pivot`，右边部分大于`pivot`。
思路：其实可以将链表拷贝到一个数组里，然后用`partition`方法，再重构链表即可，当然这样会占用线性的额外空间，我们也可以用$O\left(1\right)$的空间复杂度来实现：构建三个链表分别对应小于、等于、大于，然后将链表连起来即可，注意讨论边界情况。  

代码：  
```cpp
ListNode * ListPartition(ListNode * head, int pivot) {  
	ListNode * sh = nullptr; // 小于list  
	ListNode * sr = nullptr;  
	ListNode * eh = nullptr; // 等于list  
	ListNode * er = nullptr;  
	ListNode * bh = nullptr; // 大于list  
	ListNode * br = nullptr;  
	ListNode * next = nullptr; // save next node  
	while (head) {  
		next = head -> next;  
		head -> next = nullptr;  
		if (head -> val < pivot) {  
			if (!sh) {  
				sh = head;  
				sr = head;  
			}  
			else {  
				sr -> next = head;  
				sr = head;  
			}  
		}  
		else if (head -> val == pivot) {  
			if (!eh) {  
				eh = head;  
				er = head;  
			}  
			else {  
				er -> next = head;  
				er = head;  
			}  
		}  
		else {  
			if (!bh) {  
				bh = head;  
				br = head;  
			}  
			else {  
				br -> next = head;  
				br = head;  
			}  
		}  
		head = next;  
	}  
	if (sr) {  
		sr -> next = eh;  
		er = er ? er : sr;  
	}  
	if (er) {  
		er -> next = bh;  
	}  
    return sh ? sh : (eh ? eh : bh);  
}
```
时间复杂度$O\left(n\right)$   
空间复杂度$O\left(1\right)$  

-----

### 问题3：复制带随机指针的链表
[LeetCode138](https://leetcode-cn.com/problems/copy-list-with-random-pointer/)  
问题描述：给你一个长度为 `n` 的链表，每个节点包含一个额外增加的随机指针 `random` ，该指针可以指向链表中的任何节点或空节点。构造这个链表的 深拷贝。 深拷贝应该正好由 `n` 个 全新 节点组成，其中每个新节点的值都设为其对应的原节点的值。新节点的 `next` 指针和 `random` 指针也都应指向复制链表中的新节点，并使原链表和复制链表中的这些指针能够表示相同的链表状态。复制链表中的指针都不应指向原链表中的节点 。例如，如果原链表中有 `X` 和 `Y` 两个节点，其中 `X.random --> Y` 。那么在复制链表中对应的两个节点 `x` 和 `y` ，同样有 `x.random --> y` 。  
返回复制链表的头节点。  
思路：  
- 如果可以使用额外空间，利用哈希表实现老节点和新节点之间的对应关系，在构建新节点的`next`和`random`指针时，就能找到对应的新节点。
- 如果只用$O\left(1\right)$的空间复杂度，将新老节点的对应关系**就地**记录在链表中，即将老的链表节点复制一份并连接在它后面，这样，老节点的`random`指针指向的节点的后一个，就是新节点的`random`指针应该指向的位置，新节点的`next`指针就是新复制的节点的`next`的`next`。在同一个链表中复制了新的节点，这样就能保留下新老节点的对应关系。最后将新节点全部从老链表上取下即可。

方法1代码：  
```cpp
Node* copyRandomList(Node* head) {
	unordered_map<Node*, Node*> nmap;
	Node* p = head;
	while (p) {
		nmap[p] = new Node(p -> val);
		p = p -> next; 
	}
	p = head;
	while (p) {
		nmap[p] -> next = nmap[p -> next];
		nmap[p] -> random = nmap[p -> random];
		p = p -> next;
	}
	return nmap[head];
}
```
时间复杂度$O\left(n\right)$   
空间复杂度$O\left(n\right)$  

方法2代码：  
```cpp
Node* copyRandomList(Node* head) {
	if (!head)
	return head;
	Node * p = head, * t = nullptr;
	// copy linklist
	// 1 -> 2
	// 1 -> 1' -> 2
	while (p) {
		t = p -> next;
		p -> next = new Node(p -> val);
		p -> next -> next = t;
		p = t;
	}
	p = head;
	Node * copy = nullptr;
	// set copy node ran
	while (p) {
		copy = p -> next;
		t = copy -> next;
		copy -> random = p -> random ? p -> random -> next : nullptr;
		p = t;
	}
	Node * rethead = head -> next;
	p = head;
	// split nodelist
	while (p) {
		t = p -> next -> next;
		copy = p -> next;
		p -> next = t;
		copy -> next = t ? t -> next : nullptr;
		p = t;
	}
	return rethead;
}
```
时间复杂度$O\left(n\right)$   
空间复杂度$O\left(1\right)$  

-----

### 问题4：在一个有环单链表中找到第一个入环节点
[LeetCode142](https://leetcode-cn.com/problems/linked-list-cycle-ii/)
思路1：遍历链表，并且用哈希表记录遍历过的节点，当遍历到已经记录的节点时，该节点就是入环节点，空间复杂度为$O\left(n\right)$；  
思路2：使用快慢指针，两指针从头开始走，快指针一次走两步，慢指针一次走一步，一定会在环中的某个节点相遇，然后快指针回到头节点，此时两指针同时一次走一步，相遇时的节点即为入环节点，此方法空间复杂度为$O\left(1\right)$。  
方法2代码：  
```cpp
// getLoopNode
ListNode *getLoopNode(ListNode *head) {
	if (!head || !head -> next || !head -> next -> next) // 两个节点成环的情况也能包括
		return nullptr;
	ListNode * p1 = head -> next; // slow
	ListNode * p2 = head -> next -> next; // fast
	while (p1 != p2) {
		if (!p2 -> next || !p2 -> next -> next) // 无环
			return nullptr;
		p1 = p1 -> next;
		p2 = p2 -> next -> next;
	}
	p2 = head;
	while (p1 != p2) {
		p1 = p1 -> next;
		p2 = p2 -> next;
	}
	return p1;
}
```
时间复杂度$O\left(n\right)$   
空间复杂度$O\left(1\right)$  

-----

### 问题5：两个单链表的相交节点
相关OJ：[LeetCode160](https://leetcode-cn.com/problems/intersection-of-two-linked-lists/)
问题描述：给定两个可能有环也可能无环的单链表，头节点`head1 head2`，实现一个函数，若两链表相交，返回第一个相交节点，否则返回空。  
思路：  对于有环和无环分情况讨论
1. 两个链表都无环。遍历两个链表，如果两个链表最后一个节点不相等，那么一定不相交，否则相交，遍历时记录两个链表长度的差值，再次遍历时，长链表先多走差值步，然后一起走，找到的相同节点即为相交节点（当然也可以用哈希表）；
2. 如果一个链表有环，而另一个无环，那必然不相交；
3. 两个链表都有环。
	1. 如果两个链表的入环节点是同一个，那么第一个相交节点应该在该节点处或者之前，将该节点看作终止节点，用无环方法找第一个相交节点即可；
	2. 如果两个链表的入环节点不是同一个，那么让一个入环节点转一圈，如果没有遇到另一个入环节点，那么就不相交，否则相交，如果相交返回任意一个入环节点即可。

代码：  
```cpp
// noLoop 两无环链表返回第一个相交节点
ListNode *noLoop(ListNode *headA, ListNode *headB) {
	if (!headA || !headB)
		return nullptr;
	ListNode *p1 = headA;
	ListNode *p2 = headB;
	int n = 0; // 链表长度差值
	while (p1) {
		++ n;
		p1 = p1 -> next;
	}
	while (p2) {
		-- n;
		p2 = p2 -> next;
	}
	if (p1 != p2)
		return nullptr;
	p1 = n > 0 ? headA : headB; // p1 指向长链
	p2 = p1 == headA ? headB : headA;
	n = abs(n);
	while (n) {
		p1 = p1 -> next;
		-- n;
	}
	while (p1 != p2){
		p1 = p1 -> next;
		p2 = p2 -> next;
	}
	return p1;
}

// bothLoop 两有环链表返回第一个相交节点
ListNode *bothLoop(ListNode *head1, ListNode *loop1, ListNode *head1, ListNode *loop2) {
	// head 头节点 loop 入环节点
	ListNode *p1 = nullptr;
	ListNode *p2 = nullptr;
	if (loop1 == loop2) { // 入环节点相同
		p1 = head1;
		p2 = head2;
		int n = 0; // 链表长度差值 将loop看作end
		while (p1 != loop1) {
			++ n;
			p1 = p1 -> next;
		}
		while (p2 != loop1) {
			-- n;
			p2 = p2 -> next;
		}
		p1 = n > 0 ? headA : headB; // p1 指向长链
		p2 = p1 == headA ? headB : headA;
		n = abs(n);
		while (n) {
			p1 = p1 -> next;
			-- n;
		}
		while (p1 != p2) {
			p1 = p1 -> next;
			p2 = p2 -> next;
		}
		return p1;
	}
	else {// 入环节点不同
		p1 = loop1 -> next;
		while (p1 != loop1) { // p1 在环中转一圈 如果 loop2也在环中 则相交 返回 任一入环节点
			if (p1 == loop2)
				return loop1;
			p1 = p1 -> next;
		}
		return nullptr;
	}
}

// 主函数
ListNode *getIntersectNode(ListNode *head1, ListNode *head2) {
	if (!head1 || !head2)
		return nullptr;
	ListNode *loop1 = getLoopNode(head1);
	ListNode *loop2 = getLoopNode(head2);
	if (!loop1 && !loop2) // 两无环
		return noLoop(head1, head2);
	if (loop1 && loop2)		// 两有环
		return bothLoop(head1, loop1, head2, loop2);
	return nullptr;		// 一个有一个无
}
```
时间复杂度$O\left(n\right)$   
空间复杂度$O\left(1\right)$  
