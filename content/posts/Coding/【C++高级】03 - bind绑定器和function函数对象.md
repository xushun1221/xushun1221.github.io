---
title: 【C++高级】03 - 绑定器、函数对象
date: 2022-09-11
tags: [C++]
categories: [Coding]
---

绑定器和函数对象都是c++11从boost库中引入的机制。

## bind 绑定器

在c++ STL中提供了两个绑定器：

1. `bind1st`：将`operator()`的第一个形参变量绑定成一个确定的值；
2. `bind2nd`：将`operator()`的第二个形参变量绑定曾一个确定的值。

简单来说就是，绑定器和二元函数对象进行组合，就会得到一个一元函数对象。

### 何时使用绑定器？

示例：  
```C++
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;
template <typename Container>
void showContainer(Container& c) {
    /* 实例化之前 编译器不知道 Container::iterator 是不是类型 需要 typename 修饰 */
    typename Container::iterator it = c.begin();
    for ( ; it != c.end(); ++ it) {
        cout << *it << " ";
    }
    cout << endl;
}
int main() {
    vector<int> vec;
    srand(time(nullptr));
    for (int i = 0; i < 20; ++ i) {
        vec.push_back(rand() % 100 + 1);
    }
    showContainer(vec);
    /* sort 默认从小到大排序 */
    sort(vec.begin(), vec.end());
    showContainer(vec);
    /* 从大到小排序 需要二元函数对象 greater */
    sort(vec.begin(), vec.end(), greater<int>());
    showContainer(vec);
    /* 
        现在 vec 从大到小排序 
        我们要将 70 按序插入 需要找到第一个比 70 小的元素 
        使用 find_if 函数进行查找
        find_if 需要一个一元的函数对象  它和 70 进行比较
        而 less 和 greater 都是二元的函数对象
        这时就需要使用 bind 绑定器 进行参数绑定 使其变成一元函数对象
    */
    auto it1 = find_if(vec.begin(), vec.end(), bind2nd(less<int>(), 70));
    auto it2 = find_if(vec.begin(), vec.end(), bind1st(greater<int>(), 70));
    /*
        在 vec 中找到第一个小于70的元素
        bind2nd + less     a < 70 ?
        bind1st + greater  70 > a ?
        这两种都可以
    */
    if (it1 == it2) {
        vec.insert(it1, 70);
    }
    showContainer(vec);
    return 0;
}
```


### 绑定器的实现原理

绑定器实现的原理，其实就是：  
1. 绑定器函数接收一个二元函数对象和一个固定参数，并返回一个一元函数对象；
2. 该一元函数对象由自定义的函数对象提供；
3. 该函数对象仅仅是对原本的二元函数对象进行了封装，将其某个参数固定为一个量，它的`operator()`重载仅接收一个参数，使用上就和一元函数对象相同了。

可以看出，绑定器是函数对象的一个应用。

示例：  
```C++
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;

template <typename Container>
void showContainer(Container& c) {
    typename Container::iterator it = c.begin();
    for ( ; it != c.end(); ++ it) { cout << *it << " "; }
    cout << endl;
}

template <typename Iterator, typename Compare>
Iterator my_find_if(Iterator first, Iterator end, Compare comp) {
    /* 在 [first, end) 范围上遍历时 第一个满足 comp 的迭代器 返回 */
    for (; first != end; ++ first) {
        if (comp(*first) == true) {
            return first;
        }
    }
    return end;
}

template <typename Compare, typename T>
class _mybind1st {
    /* 一元函数对象的类 */
public:
    _mybind1st(Compare comp, T val) : _comp(comp), _val(val) {}
    /* operator() 重载 只需要第二个参数 第一个由绑定器提供了 */
    bool operator()(const T& second) {
        /* 仅仅是对 comp 函数对象进行了封装 底层仍然是 comp 的执行 */
        return comp(_val, second);
    }
private:
    Compare _comp;
    T _val;
};

template <typename Compare, typename T>
_mybind1st<Compare, T> my_bind1st(Compare comp, const T& val) {
    /* 
        my_bind1st 接收二元函数对象和一个值 
        将 该值和二元函数对象的第一个参数绑定起来
        并 返回一个一元函数对象
    */
    return _mybind1st<Compare, T>(comp, val);
}

int main() {
    vector<int> vec;
    srand(time(nullptr));
    for (int i = 0; i < 20; ++ i) {
        vec.push_back(rand() % 100 + 1);
    }
    showContainer(vec);
    sort(vec.begin(), vec.end(), greater<int>());
    showContainer(vec);
    /* 使用自己写的绑定器 */
    auto it = find_if(vec.begin(), vec.end(), bind1st(greater<int>(), 70));
    vec.insert(it, 70);
    showContainer(vec);
    return 0;
}
/*
xushun@xushun-virtual-machine:~/cppadvanced$ g++ bindtest2.cpp && ./a.out
41 17 78 62 90 34 65 91 90 100 88 22 21 33 50 39 39 52 53 16 
100 91 90 90 88 78 65 62 53 52 50 41 39 39 34 33 22 21 17 16 
100 91 90 90 88 78 70 65 62 53 52 50 41 39 39 34 33 22 21 17 16 
*/
```



## function 函数对象类型

要使用`function`类，需要包含头文件`functional`。













