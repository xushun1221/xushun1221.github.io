---
title: 【C++基础】08 - STL组件
date: 2022-08-27
tags: [C++, STL]
categories: [Coding]
---

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


## 顺序容器

### vector 容器

vector：向量容器，头文件`vector`。

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


### deque 容器

deque：双端队列容器，头文件`deque`。

底层的数据结构：动态开辟的二维数组。第一维初始值为2（一开始有2行），扩容时以2倍的方式进行扩容（2行变4行），扩容后，原来第二维的数组从新的二维数组的第一位下标`oldsize / 2`的位置开始存放（原内容复制到二维数组的第1、2行，从0行开始），上下都预留相同的空行，方便支持deque首尾元素的添加。

注意，这里的*二维数组*并不是严格意义上的二维数组，而是类似拉链法的哈希表的结构，第二维的数组（行）是独立的。第二维在内存上是**分段连续**的。而且，二维数组的**第二维的大小是固定的**（行的长度是固定的），扩容是在第一维度上进行的。

deque底层结构示意图：

![](/post_images/posts/Coding/C++基础/deque起始情况.jpg "deque起始情况")

![](/post_images/posts/Coding/C++基础/deque添加元素.jpg "deque添加元素")

![](/post_images/posts/Coding/C++基础/deque扩展.jpg "deque扩展")


元素的操作：  
- 增加：
  - `push_back(t)`，在容器末尾添加元素，时间复杂度$O\left(1\right)$，可能导致容器的扩容；
  - `push_front(t)`，在容器首部添加元素，时间复杂度$O\left(1\right)$，可能导致容器的扩容；
  - `insert(it, t)`，在`it`迭代器指向的位置添加一个元素，时间复杂度$O\left(n\right)$，可能导致容器的扩容。
- 删除：
  - `pop_back()`，删除容器末尾的元素，时间复杂度$O\left(1\right)$；
  - `pop_front()`，删除容器首元素，时间复杂度$O\left(1\right)$；
  - `erase(it)`，删除`it`迭代器指向的元素，时间复杂度$O\left(n\right)$。
- 查询：  
  - 通过`iterator`进行遍历查询。

注意，`insert`和`erase`也要考虑迭代器失效问题。



### list 容器

list：链表容器，头文件`list`。

底层数据结构：双向循环链表。

元素的操作：  
- 增加：
  - `push_back(t)`，在容器末尾添加元素，时间复杂度$O\left(1\right)$；
  - `push_front(t)`，在容器首部添加元素，时间复杂度$O\left(1\right)$；
  - `insert(it, t)`，在`it`迭代器指向的位置添加一个元素，时间复杂度$O\left(1\right)$。
- 删除：
  - `pop_back()`，删除容器末尾的元素，时间复杂度$O\left(1\right)$；
  - `pop_front()`，删除容器首元素，时间复杂度$O\left(1\right)$；
  - `erase(it)`，删除`it`迭代器指向的元素，时间复杂度$O\left(1\right)$。
- 查询：  
  - 通过`iterator`进行遍历查询。

注意，list中`insert`不会导致迭代器失效，`erase(it)`会导致当前位置`it`的迭代器失效，而该容器的其他迭代器不会受到影响。



### 顺序容器的对比（vector、deque、list）

- vector特点：动态数组，连续内存，2倍扩容；
- deque特点：动态二维数组，第二维固定，第一维进行扩容，第二维大小固定、分段连续；
- list特点：双向循环链表。

#### 常见问题

deque底层的数据结构是否使用连续的内存？  
答： 不是连续的内存，deque的第二维在内存上是分段连续的，每个第二维的行是连续的，但整体不是连续的。


vector与deque之间的区别？  
答：  
1. 底层数据结构是不同的，vector使用一块连续的内存，而deque是分段连续的；
2. 元素插入和删除的时间复杂度不同：
   1. deque在首尾进行元素插入和删除的时间复杂度为$O\left(1\right)$；
   2. vector在首部进行插入删除的时间复杂度为$O\left(n\right)$，而在尾部是$O\left(1\right)$；
   3. deque和vector在容器中间进行元素插入和删除的时间复杂度都是$O\left(n\right)$。
3. 内存的使用效率方面：
   1. vector必须使用连续的内存空间；
   2. deque无需使用连续的内存。
4. 在容器中部进行元素的插入和删除操作，vector和deque的效率谁高？
   1. 它们两的时间复杂度都是$O\left(n\right)$，但是vector的效率要更高一些；
   2. 因为vector的内存是完全连续的，插入删除时只需依次对元素进行移位操作即可；
   3. deque是分段连续的，在两个不同的第二维数组（行）中移动元素时，需要首先访问第一维，找到目标行的首地址，才能进行元素的移动，效率要稍微慢一些。


vector和list的区别？（数组和链表的区别）
答：  
1. 底层数据结构不同：
   1. vector：数组（连续内存空间）；
   2. list：双向循环链表（无需连续内存空间）。
2. vector支持随机访问，而list的查询复杂度为$O\left(1\right)$；
3. vector插入和删除的复杂度为$O\left(n\right)$，而list为$O\left(1\right)$。






## 容器适配器 adaptors

什么是容器适配器？  
1. 适配器底层没有自己的数据结构，它是另外一个容器的封装，它的方法，全部依赖于底层的容器实现；
2. 没有实现自己的迭代器，不能用迭代器来遍历适配器。

### stack 容器适配器
















