# 【算法】C++对数器


## 对数器是什么
对数器是用来测试算法正确性的一种方式，在找不到合适的在线OJ时，我们也可以编写一个对数器来测试自己编写的算法的正确性。原理是用一个绝对正确的算法和你自己的算法，对同一批数据进行测试，如果在数据量较大时，两种算法的结果仍然相同，那么就可以认为，我们的算法是正确的。

-----

## 对数器的组成
1. 待测试的算法
2. 绝对正确即使复杂度较高的算法
3. 随机样本生成器
4. 对比算法输出的方法
5. 测试结果的输出方法

-----

## C++对数器实现
简单用一个`BubbleSort`和STL自带的`sort`来写一个对数器的示例。

用到的头文件
```C++
#include <vector>  
#include <algorithm>  
#include <assert.h>  
#include <time.h>  
#include <iostream>  
using namespace std;
```

自己写的排序算法
```C++
// swap two num in a vector  
template<typename T>  
void swapNum(vector<T>& vec, int i, int j) {  
	T temp = vec[i];  
	vec[i] = vec[j];  
	vec[j] = temp;  
}  
  
// my algorithm bubbleSort  
template<typename T>  
void bubbleSort(vector<T>& vec) {  
	int len = vec.size();  
	if (len < 2)  
		return;  
	for (int i = len - 1; i > 0; -- i) {  
		bool done = true;  
		for (int j = 0; j < i; ++ j) {  
			if (vec[j] > vec[j + 1]) {  
				swapNum(vec, j, j + 1);  
				done = false;  
			}  
		}  
		if (done)  
			return;  
	}
}
```

STL的排序算法
```C++
// compare method  
template<typename T>  
bool compare(T a, T b) {  
	return a < b;  
}  
  
// absolute right algorithm using STL  
template<typename T>  
void rightMethod(vector<T>& vec) {  
	sort(vec.begin(), vec.end(), compare<T>);  
}
```

随机vector生成器
```C++
// generate random vector  
template<typename T>  
void ranVecGenerator(vector<T>& vec, int maxSize, T minValue, T maxValue) {  
	assert(maxValue > minValue); // assert.h  
	srand((unsigned int)time(nullptr)); // time.h  
	int size = rand() % maxSize + 1; // [1, maxSize]  
	vec = vector<T>(size, -1);  
	for (auto& x : vec)  
		x = rand() % (maxValue - minValue) + minValue + 1; // [minValue + 1, maxValue]  
}
```

复制、比较、显示函数
```C++
// copy a vector to another  
template<typename T>  
void copyVec(vector<T>& vecIn, vector<T>& vecOut) {  
    vecIn = vecOut;  
}  
  
// compare two vector  
template<typename T>  
bool isEqual(vector<T>& vec1, vector<T>& vec2) {  
    if (vec1.size() != vec2.size())  
        return false;  
 	else 
		return equal(vec1.begin(), vec1.end(), vec2.begin());  
 	return false;
}  
  
// display vector  
template<typename T>  
void displayVec(vector<T>& vec) {  
	for (auto x : vec)  
		cout << x << " ";  
	cout << endl;  
}
```

测试算法正确性的函数
```C++
// test my algorithm using right method  
template<typename T>  
void testVecAlgorithm(int testEpoch, int maxSize, int minValue, int maxValue, void (*rightFunc)(vector<T>&), void (*testFunc)(vector<T>&)) {  
	bool Accept = true;  
	clock_t startTime, endTime;  
	startTime = clock();  
	for (int i = 0; i < testEpoch; ++ i) {  
		vector<T> vec1, vec2, vec3;  
		ranVecGenerator(vec1, maxSize, minValue, maxValue);  
		copyVec(vec2, vec1);  
		copyVec(vec3, vec1);  
		rightFunc(vec1);  
		testFunc(vec2);  
		if (!isEqual(vec1, vec2)) {  
			Accept = false;  
			displayVec(vec3);  
			break; 
		}  
		vector<T>().swap(vec1);  
		vector<T>().swap(vec2);  
		vector<T>().swap(vec3);  
	}  
	endTime = clock();  
	if (Accept)  
		cout << "ACCEPT!" << endl;  
	else 
		cout << "WRONG!" << endl; 
	cout << "Time using: " << (endTime - startTime) / double(CLOCKS_PER_SEC) << 's' << endl; 
}
```

测试
```C++
int main() {  
	testVecAlgorithm(100, 100, -101, 100, bubbleSort<int>, rightMethod<int>);  
	return 0;  
}
```

输出：
![](/post_images/posts/Coding/算法——C++对数器/C++对数器运行结果.jpg "对数器运行结果")


