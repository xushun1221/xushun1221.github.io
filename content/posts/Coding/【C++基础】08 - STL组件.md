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


### stack

stack：栈，FILO，使用deque作为默认底层容器。

方法：  
- `push(t)`，入栈；
- `pop()`，出栈；
- `top()`，查看栈顶元素；
- `empty()`，判断栈空；
- `size()`，返回元素个数。 


### queue

queue：队列，FIFO，使用deque作为默认底层容器。

方法：  
- `push(t)`，入队；
- `pop()`，出队；
- `front()`，队头元素；
- `back()`，队尾元素；
- `empty()`，判断栈空；
- `size()`，返回元素个数。

为什么stack和queue使用deque作为底层容器，而不是vector？  
答：  
1. vector初始时的内存使用效率太低了，vector初始时数组内存大小：`0->1->2->4->8...`，需要频繁地进行数组扩容操作，而deque初始时就提供了较大的内存：可以存储`4096 / sizeof(type)`个元素；
2. queue需要支持尾部插入和头部删除的操作，尾部插入操作对于vector和deque来说是一样的$O\left(1\right)$，而头部删除操作，vector需要$O\left(n\right)$，而deque只有$O\left(1\right)$，效率更高；
3. vector需要大片的连续内存，而deque只需要分段连续的内存，当存储大量数据时，deque对内存的利用率更高，不易产生大量的内存碎片。


### priority_queue

priority_queue：优先队列，使用vector作为底层容器。

优先队列底层将数据组织成一个**堆**结构，默认是大根堆。

方法：  
- `push(t)`，添加元素；
- `top()`，堆顶元素；
- `pop()`，弹出堆顶元素；
- `empty()`，判断堆空；
- `size()`，返回元素个数。

为什么priority_queue使用vector作为底层容器？  
答：  堆结构需要计算元素下标之间的关系，需要在一块内存连续的数组上进行组织。


## 关联容器 Associative containers

关联容器分为有序关联容器和无序关联容器两种。

set：集合，key  
map：映射表，[key, value]

- 无序关联容器(unordered)
  - 底层结构：哈希表
  - 增删查复杂度：$O\left(1\right)$
  - `unordered_set`
  - `unordered_multiset` （`multi`表示可以存在多个重复key）
  - `unordered_map`
  - `unordered_multimap`
  - 头文件`unordered_set`，`unordered_map`
- 有序关联容器
  - 底层结构：红黑树
  - 增删查复杂度：$O\left(logn\right)$
  - `set`
  - `multiset`
  - `map`
  - `multimap`
  - 头文件`set`，`map`


### unordered_set unordered_multiset

`unordered_set unordered_multiset`示例：  
```C++
#include <iostream>
#include <unordered_set>
using namespace std;

int main() {
    /* unordered_set 的 insert 和 count 操作 */
    unordered_set<int> set1; // unordered_set 中的key不能重复
    for (int i = 0; i < 50; ++ i) {
        set1.insert(rand() % 10 + 1); // insert 操作
    }
    cout << set1.size() << endl << set1.count(5) << endl; // count 操作  

    unordered_multiset<int> set2;
    for (int i = 0; i < 50; ++ i) {
        set2.insert(rand() % 10 + 1); // insert 操作
    }
    cout << set2.size() << endl << set2.count(5) << endl; // count 操作  

    /* unordered_set 使用迭代器遍历 */
    for (auto it1 = set1.begin(); it1 != set1.end(); ++ it1) {
        cout << *it1 << " ";
    }
    cout << endl;
    for (auto it2 = set2.begin(); it2 != set2.end(); ++ it2) {
        cout << *it2 << " ";
    }
    cout << endl;

    /* unordered_set 的 erase 操作 */
    set2.erase(1); // erase(key)  unordered_multiset 中会把key相同的元素全部删除
    for (auto it = set2.begin(); it != set2.end(); ) {
        if (*it == 2) {
            it = set2.erase(it); // erase(it) 只会删除当前it指向的元素 且迭代器存在失效问题
        } else {
            ++ it;
        }
    }
    for (auto it = set2.begin(); it != set2.end(); ++ it) {
        cout << *it << " ";
    }
    cout << endl;

    /* unordered_set find 操作 */
    auto it1 = set1.find(5); // find(key) 操作 存在返回指向元素的迭代器
    if (it1 != set1.end()) { // 不存在返回 end() 迭代器
        set1.erase(it1);
    }
    for (auto v : set1) {
        cout << v << " ";
    }
    cout << endl;

    return 0;
}
```

### unordered_map unordered_multimap

`unordered_map`示例：  
```C++
#include <iostream>
#include <unordered_map>
using namespace std;

int main() {
    /*
        [key, value]
        struct pair {
            first  => key
            second => value
        }
        map 中存储的是 pair
    */
    unordered_map<int, string> map1;
    // insert 操作
    map1.insert(make_pair(1000, "张三")); // make_pair(key, value) 创建一个pair
    map1.insert({1001, "李四"});    // 或者直接构造 pair 对象 {key, value}
    map1.insert({1002, "王五"});
    cout << map1.size() << endl;
    map1.insert({1000, "张三2"}); // unordered_map 不允许key重复 插入失败
    cout << map1.size() << endl;
    
    // operator[]
    cout << map1[1000] << endl; // map[key] 查询value  unordered_map 提供了operator[]重载  multimap 没有
    map1[2000]; // 注意 使用 map[key] 时 如果map中没有key 会添加insert({key, string()})
    cout << map1.size() << endl;
    cout << map1[2000] << endl; // 空字符串
    // operator[] 会返回 value 的引用 可以进行修改
    map1[2000] = "两千";
    cout << map1[2000] << endl;

    // erase
    map1.erase(2000); // erase(key)
    cout << map1.size() << endl;

    // find 迭代器访问
    auto it = map1.find(1002); // 存在 返回指向元素的迭代器
    if (it != map1.end()) { // 不存在key 返回 end()
        cout << it->first << ": " << it->second << endl; // 使用 first second 访问 key value
    }
    for (auto it = map1.begin(); it != map1.end(); ++ it) {
        cout << it->first << ": " << it->second << endl;
    }
    return 0;
}
```

### set map

示例：  
```C++
#include <iostream>
#include <set>
#include <map>
using namespace std;

class Stu {
public:
    Stu(int id = -1, string name = "none") : _id(id), _name(name) {}
    bool operator<(const Stu& stu) const { // 提供小于运算符的重载函数
        return _id < stu._id;
    }
private:
    int _id;
    string _name;
    friend ostream& operator<<(ostream& out, const Stu& stu);
};
ostream& operator<<(ostream& out, const Stu& stu) {
    out << stu._id << ": " << stu._name;
    return out;
}

int main() {
/* set */
    set<int> set1; // set 底层的红黑树 会维持元素的有序排列(默认小到大)
    for (int i = 0; i < 20; ++ i) {
        set1.insert(rand() % 20 + 1); // insert 操作
    }
    for (int k : set1) { // 红黑树中序遍历
        cout << k << " ";
    }
    cout << endl << set1.size() << endl; // 不允许重复 key
    // set 默认使用 < 方法进行比较 从小到大排列
    // 使用自定义类型的set时 需要为其添加小于运算的重载函数
    set<Stu> set2;
    set2.insert({1003, "张三"});
    set2.insert({1000, "李四"});
    set2.insert({1001, "王五"});
    set2.insert({1002, "小明"});
    for (auto s : set2) {
        cout << s << endl;
    }
/* map */
    map<int, Stu> map1; // 这里使用int 作为key 不需要运算符的重载
    map1.insert({1003, {1003, "张三"}});
    map1.insert({1000, {1000, "李四"}});
    map1.insert({1001, {1001, "王五"}});
    map1.insert({1002, {1002, "小明"}});
    for (auto p : map1) {
        cout << p.first << ": " << p.second << endl;
    }
    cout << map1[1001] << endl;
    return 0;
}
```



## 容器的迭代器 iterator













