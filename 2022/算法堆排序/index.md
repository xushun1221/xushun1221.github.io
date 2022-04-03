# 【算法】堆排序


## 堆排序
原理：利用大根堆（小根堆）结构来进行排序。  

### 实现1
这种实现方法，在建立最大堆时，从后向前对每个有孩子的结点进行了一次向下调整算法，建堆时间为$O\left(n\right)$，之后有`len - 1`次向下调整，每次调整时间复杂度为$O\left(logn\right)$，所以时间复杂度为$O\left(nlogn\right)$。  
代码：  
```cpp
void heapAdjustDown(vector<int>& nums, int root, int end)  
{  
	int left = root * 2 + 1;  
	while (left <= end)  
	{  
		int largest = left + 1 <= end && nums[left + 1] > nums[left] ? left + 1 : left;  
		largest = nums[largest] > nums[root] ? largest : root;  
		if (largest == root)  
			break;  
		std::swap(nums[largest], nums[root]);  
		root = largest;  
		left = root * 2 + 1;  
	}  
}  
void heapSort1(vector<int>& nums)  
{  
	int len = nums.size();  
	for (int i = (len - 1) / 2; i >= 0; -- i) // build heap  
		heapAdjustDown(nums, i, len - 1);  
	for (int i = len - 1; i > 0; -- i)  
	{  
		std::swap(nums[0], nums[i]);  
		heapAdjustDown(nums, 0, i - 1);  
	}  
}
```
时间复杂度：$O\left(nlogn\right)$  
空间复杂度：$O\left(1\right)$  

### 实现2
这个实现方法和第一种，算法整体复杂度不变，只在建立最大堆时不同，建堆时间为$O\left(nlogn\right)$，所以略逊于方法1。  
代码：  
```cpp
void heapAdjustUp(vector<int>& nums, int index)  
{  
	while (nums[index] > nums[(index - 1) / 2])  
	{  
		std::swap(nums[index], nums[(index - 1) / 2]);  
		index = (index - 1) / 2;  
	}  
}  
void heapAdjustDown(vector<int>& nums, int root, int end)  
{  
	int left = root * 2 + 1;  
	while (left <= end)  
	{  
		int largest = left + 1 <= end && nums[left + 1] > nums[left] ? left + 1 : left;  
		largest = nums[largest] > nums[root] ? largest : root;  
		if (largest == root)  
			break;  
		std::swap(nums[largest], nums[root]);  
		root = largest;  
		left = root * 2 + 1;  
	}  
}  
void heapSort2(vector<int>& nums)  
{  
	int len = nums.size();  
	for (int i = 0; i < len; ++ i)  
		heapAdjustUp(nums, i);  
	for (int i = len - 1; i > 0; -- i)  
	{  
		std::swap(nums[0], nums[i]);  
		heapAdjustDown(nums, 0, i - 1);  
	}  
}
```
时间复杂度：$O\left(nlogn\right)$  
空间复杂度：$O\left(1\right)$  

-----

## 相关题目

### 题目1：排序一个几乎有序的数组
题目描述：已知一个几乎有序的数组，几乎有序是指，如果将数组排序，那么每个元素移动的距离不超过`k`，且相对于数组来说`k`比较小。选择合适的算法排序该数组（非递减）。  
思路：（假设`k == 6`）我们从头开始取7个元素，构成一个最小堆，那么堆顶元素一定是这7个元素里最小的，放在数组0位置，然后将第8个元素放入堆顶，调整成最小堆，那么数组1位置就是堆顶元素，……，循环往复直到最小堆对应数组最后7个空位，依次将堆顶元素放入数组，即可。  
时间复杂度分析，建立最小堆需要常数时间，每次弹出堆顶元素，重新调整最小堆需要$O\left(logk\right)$，整体时间复杂度为$O\left(nlogk\right)$，在`k`较小时可以看作$O\left(n\right)$。  

*最小堆的实现我们可以自己写，也可以用STL方法，STL实现的优先队列，控制粒度不如自己写，效率可能较低，这题没什么差别。*    
代码：  
```cpp
void sortVecDistanceLessK(std::vector<int>& nums, int k)  
{  
	std::priority_queue<int, std::vector<int>, std::greater<int>> min_heap;  
	int index = 0, len = nums.size();  
	for (; index < std::min(len, k); ++ index)  
		min_heap.push(nums[index]);  
	int i = 0;  
	for (; index < len; ++ i, ++ index)  
	{  
		min_heap.push(nums[index]);  
		nums[i] = min_heap.top();  
		min_heap.pop();  
	}  
	while (!min_heap.empty())  
	{  
		nums[i ++] = min_heap.top();  
		min_heap.pop();  
	}  
}
```
时间复杂度：$O\left(n\right)$  
空间复杂度：$O\left(1\right)$  

### C++优先队列
定义：`priority_queue<Type, Container, Functional>`，Type 就是数据类型，Container 就是容器类型（Container必须是用数组实现的容器，比如vector,deque等等，但不能用 list。STL里面默认用的是vector），Functional 就是比较的方式。默认情况下，是大根堆，就是递减排序。  
用法：  
```cpp
//升序队列
priority_queue <int,vector<int>,greater<int>> q;
//降序队列
priority_queue <int,vector<int>,less<int>> q;

//greater和less是std实现的两个仿函数（就是使一个类的使用看上去像一个函数。其实现就是类中实现一个operator()，这个类就有了类似函数的行为，就是一个仿函数类了）
```

-----

### 题目2：数据流中的中位数
[LeetCode295](https://leetcode-cn.com/problems/find-median-from-data-stream/)  
题目描述：在一个数据流中，可以随时取得中位数。  
思路：维护两个堆结构，一个大根堆，一个小根堆，大根堆用来存储较小的一半元素，小根堆用来存储较大的一半元素，大根堆堆顶元素即为中位数，当心数据到来时，要做的是：  
1. 如果比大根堆堆顶元素小，入大根堆，否则入小根堆
2. 如果两堆元素数差值超过1，调整两边的大小，大根堆多就将堆顶元素入小根堆，反之亦然

代码：  
```cpp
class MedianFinder {
public:
    priority_queue<int, vector<int>, greater<int>> min_heap;
    priority_queue<int, vector<int>, less<int>> max_heap;

    MedianFinder() {}
    
    void addNum(int num) {
        if (max_heap.empty()) {
            max_heap.push(num);
            return;
        }
        if (num <= max_heap.top()) {
            max_heap.push(num);
            if (max_heap.size() - min_heap.size() > 1) {
                min_heap.push(max_heap.top());
                max_heap.pop();
            }
        }
        else {
            min_heap.push(num);
            if (min_heap.size() - max_heap.size() > 1) {
                max_heap.push(min_heap.top());
                min_heap.pop();
            }
        }
    }
    
    double findMedian() {
        if (max_heap.size() == min_heap.size())
            return (double(max_heap.top()) + double(min_heap.top())) / 2;
        if (max_heap.size() > min_heap.size())
            return max_heap.top();
        return min_heap.top();
    }
};
```

-----

### 题目3：滑动窗口中位数
[LeetCode480](https://leetcode-cn.com/problems/sliding-window-median/)  
题目描述：给你一个数组 `nums`，有一个长度为 `k` 的窗口从最左端滑动到最右端。窗口中有 `k` 个数，每次窗口向右移动 1 位。你的任务是找出每次窗口移动后得到的新窗口中元素的中位数，并输出由它们组成的数组。  
思路：很容易想到，我们需要维持一个这样的数据结构来找中位数：  
1. 可以`insert(int num)`
2. 可以`erase(int num)`
3. 可以`getMiddle()`

由上一题，我们可以用双堆的结构来在数据流中快速地找到中位数，但是堆结构不能删除非堆顶元素，所以需要用**延迟删除**方法来删除堆中的非堆顶元素。在该题中，我们维持一个哈希表，记录需要删除的元素以及要删除的次数，并且用两个整型变量来记录双堆的逻辑大小（不能用`size()`函数计算，因为延迟删除只是在逻辑上删除，而仍存在堆中）。  
那么，应该何时删除要删除的元素呢？答案是，当要删除的元素出现在堆顶时，删除它。在进行双堆大小调整时，可能会将待删除元素暴露在堆顶，此时删除即可。  
代码：  
```cpp
class DualHeap {
private:
    priority_queue<int, vector<int>, less<int>> small;      // max heap
    priority_queue<int, vector<int>, greater<int>> large;   // min heap
    unordered_map<int, int> delayed;
    int smallSize;
    int largeSize;
    // 弹出栈顶的待删除元素
    template<typename T>
    void prune(T& heap) {
        while (!heap.empty()) {
            int t = heap.top();
            if (delayed.count(t)) {
                heap.pop();
                -- delayed[t];
                if (!delayed[t])
                    delayed.erase(t);
            }
            else
                break;
        }
    }
    // 维持双堆平衡
    void makeBalance() {
        if (smallSize > largeSize + 1) {
            large.push(small.top());
            ++ largeSize;
            small.pop();
            -- smallSize;
            prune(small); // small堆顶元素改变 需要prune
        }
        else if (largeSize > smallSize) {
            small.push(large.top());
            ++ smallSize;
            large.pop();
            -- largeSize;
            prune(large); // large堆顶元素改变 需要prune

        }
    }

public:
    DualHeap() : smallSize(0), largeSize(0) {}
    // 插入新元素
    void insert(int num) {
        if (small.empty() || num <= small.top()) {
            small.push(num);
            ++ smallSize;
        }
        else {
            large.push(num);
            ++ largeSize;
        }
        makeBalance();
    }
    // 删除指定元素
    void erase(int num) {
        ++ delayed[num];
        if (num <= small.top()) {
            -- smallSize;
            if (num == small.top()) 
                prune(small);
        }
        else {
            -- largeSize;
            if (num == large.top())
                prune(large);
        }
        makeBalance();
    }
    // 获得中位数
    double getMiddle() {
        if (smallSize > largeSize)
            return small.top();
        else
            return (double(small.top()) + double(large.top())) / 2;
    }
};


class Solution {
public:
    vector<double> medianSlidingWindow(vector<int>& nums, int k) {
        vector<double> res;
        DualHeap dualheap;
        for (int i = 0; i < k; ++ i) 
            dualheap.insert(nums[i]);
        res.push_back(dualheap.getMiddle());
        for (int i = k; i < nums.size(); ++ i) {
            dualheap.insert(nums[i]);
            dualheap.erase(nums[i - k]);
            res.push_back(dualheap.getMiddle());
        }
        return res;
    }
};
```

-----
