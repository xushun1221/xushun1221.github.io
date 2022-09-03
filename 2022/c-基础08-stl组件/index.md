# 【C++基础】08 - STL组件


## Standard Template libaray 标准模板库

学习内容：  
1. 标准容器
   1. 顺序容器
      1. vector
      2. deque
      3. list
   2. 容器适配器
      1. stack
      2. queue
      3. priority_queue
   3. 关联容器
      1. 无需关联容器
         1. unordered_set
         2. unordered_multiset
         3. unordered_map
         4. unordered_multimap
      2. 有序关联容器
         1. set
         2. multiset
         3. map
         4. multimap
2. 近容器
   1. 数组
   2. string
   3. bitset
3. 迭代器
   1. iterator
   2. const_iterator
   3. reverse_iterator
   4. const_reverse_iterator
4. 函数对象
   1. greater
   2. less
5. 泛型算法
   1. sort
   2. find
   3. find_if
   4. binary_search
   5. for_each


## vector 容器

向量容器，头文件`vector`。

底层的数据结构：动态开辟的数组，每次以原来空间大小的2倍进行扩容。

元素的操作：  
- 增加：
  - `push_back(t)`，容器末尾添加元素，时间复杂度$O\left(1\right)$，可能导致容器的扩容；
  - `insert(it, t)`，在迭代器`it`指向的位置插入一个元素，时间复杂度$O\left(n\right)$，也可能导致容器的扩容。
- 删除：
  - `pop_back()`，删除末尾的元素，时间复杂度$O\left(1\right)$；
  - `erase(it)`，删除迭代器`it`指向的元素，时间复杂度$O\left(n\right)$。
- 查询：
  - `operator[]`，中括号运算符重载，支持下标的随机访问，时间复杂度$O\left(1\right)$；
  - 通过`iterator`迭代器进行遍历查询；
  - 通过泛型算法`find`、`for_each`等进行遍历查询。

注意，`insert`和`erase`操作需要对迭代器进行更新，否则操作过的迭代器及该容器后面的迭代器会失效。（之前写过迭代器失效问题）

其他常用方法：  
- `size()`，元素个数；
- `empty()`，是否为空；
- `reserve(int)`，为容器预留空间，只会分配内存空间，但是并不会添加新的元素；
- `resize(int)`，为容器扩容，不仅会分配内存空间，还会田间新的元素。


使用示例：  
```C++
#include <iostream>
#include <vector>
using namespace std;

int main() {

    vector<int> vec; // 起始时元素空间为0
    cout << vec.size() << ", " << vec.capacity() << endl;
    vec.reserve(20); // 直接让容器分配20个元素的内存空间 避免多次扩容 但是不会添加新的元素
    // vec.resize(20); // resize 不仅会分配内存空间 也会添加新的元素(0)
    cout << vec.size() << ", " << vec.capacity() << endl;

    for (int i = 0; i < 20; ++ i) {
        vec.push_back(rand() % 100 + 1);
    }
    cout << vec.size() << ", " << vec.capacity() << endl;

    for (int i = 0; i < vec.size(); ++ i) {
        cout << vec[i] << " "; // operator[] 重载 访问
    }
    cout << endl;

    // 删除vec中所有的偶数
    auto it2 = vec.begin();
    while (it2 != vec.end()) {
        if (*it2 % 2 == 0) {
            it2 = vec.erase(it2); // erase 需要更新迭代器
        } else {
            ++ it2;
        }
    }

    vector<int>::iterator it1 = vec.begin();
    for (; it1 != vec.end(); ++ it1) {
        cout << *it1 << " "; // 使用迭代器解引用访问
    }
    cout << endl;

    // 在所有奇数前添加一个小一的数
    for (it1 = vec.begin(); it1 != vec.end(); ++ it1) {
        if (*it1 % 2 == 1) {
            it1 = vec.insert(it1, *it1 - 1); // insert 需要更新迭代器
            ++ it1; // 注意逻辑
        }
    }

    for (it1 = vec.begin(); it1 != vec.end(); ++ it1) {
        cout << *it1 << " ";
    }
    cout << endl;

    return 0;
}
```


## deque 和 list

