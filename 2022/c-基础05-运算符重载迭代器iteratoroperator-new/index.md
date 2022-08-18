# 【C++基础】05 - 运算符重载、迭代器iterator、operator new


运算符重载的意义在于，使得对象的运算表现得和编译器内置的类型一样。例如：  
```C++
template<typename T>
T sum(T a, T b) {
    return a + b; // a.+(b);
}
```
如果`T`是内置类型，`a + b`可以直接执行，而如果`T`是对象，那么编译器就不知道如何处理两个对象的加法运算，这时就需要对`T`的加法运算进行**运算符重载**。看似两个对象进行运算，其实是调用了成员方法。


## 实现 complex 类
下面的示例通过实现一个`complex`复数类来学习如何进行运算符重载。

包含以下内容：  
- 成员方法实现运算符重载
- 全局函数实现运算符重载，以及友元声明
- 单目运算符的重载实现，前置和后置
- 复合运算符的重载实现
- 重载`<<`以实现标准输出
- 重载`>>`以实现标准输入

```C++
#include <iostream>

class complex {
public:
    complex(int real = 0, int image = 0) : _real(real), _image(image) {}
    void show() {
        std::cout << "(real: " << _real << ", image: " << _image << ")" << std::endl;
    }
    complex operator+(const complex& src) { // 加法运算符重载
        // complex tmp; // 加数和被加数不变
        // tmp._real = _real + src._real;
        // tmp._image = _image + src._image;
        // return tmp;
        return complex(_real + src._real, _image + src._image);
        // 有个小问题：为什么可以访问src的私有成员
        // 因为封装是编译阶段的概念，是针对类型而非对象的
        // 在类的成员函数中可以访问同类型实例对象的私有成员
    }
    // 类外函数需要访问类私有成员 需要声明为友元函数
    friend complex operator+(const complex& lhs, const complex& rhs);
    complex& operator++() { // 前置自增运算符重载
        ++ _real;
        ++ _image;
        return *this; // 前置返回运算后的值
    }
    complex operator++(int) { // 后置自增运算符重载 用int标识（单目运算符的写法）
        // complex tmp = *this; // 后置返回原值
        // _real ++;
        // _image ++;
        // return tmp; // 返回值不能引用
        return complex(_real ++, _image ++); // 效率更高
    }
    void operator+=(const complex& src) { // 复合运算符
        _real += src._real;
        _image += src._image;
    }
    // << 重载的友元声明
    friend std::ostream& operator<<(std::ostream& out, const complex& comp);
private:
    int _real;
    int _image;
};

// 全局域中的重载函数
complex operator+(const complex& lhs, const complex& rhs) {
    return complex(lhs._real + rhs._real, lhs._image + rhs._image);
}

// 为了支持cout打印 重载 << 运算符
// 不能重载为成员方法 所以在全局域中实现
std::ostream& operator<<(std::ostream& out, const complex& comp) {
    out << "(real: " << comp._real << ", image: " << comp._image << ")";
    return out; // 返回输出流 支持连续输出
}

int main() {
    complex comp1(10, 10);
    complex comp2(20, 20);
    complex comp3 = comp1 + comp2; // comp1.+(comp2) 加法重载
    comp3.show();

    complex comp4 = comp1 + 20; // 这样写可以吗？
    comp4.show();
    // 可以的 实际上是 comp1.operator+(20)
    // 形参是 const complex& 实参是 20
    // 编译器就会尝试将 int 转为 complex 临时对象
    // 存在构造函数支持 complex(20, int = 0)
    
    complex comp5 = 30 + comp1;
    // 编译器执行对象运算时 会调用对象的运算符重载 优先成员方法
    // 如果没有 在全局作用域寻找合适的重载函数
    // 30 无法调用 complex 的成员方法
    // 所以需要定义全局的重载函数 complex operator+(const complex&, const complex&) 
    // 存在构造函数complex(30, int = 0) int 转为 complex 临时对象
    // 注意全局域中的函数不能访问形参的私有成员 需要在类中声明其为友元函数
    comp5.show();

    comp5 = comp1 ++;
    comp5.show();
    comp1.show();
    comp5 = ++ comp1;
    comp5.show();
    comp1.show();

    comp1 += comp2; // comp1.+=(comp2) 或 全局
    comp1.show();

    std::cout << comp1 << std::endl;
    std::cout << comp2 << " " << comp3 << std::endl;
    return 0;
}
```



## 实现 string 类

```C++
#include <iostream>
#include <cstring>
class string {
public:
    string(const char* pstr = nullptr) {
        if (pstr != nullptr) {
            _pstr = new char[strlen(pstr) + 1];
            strcpy(_pstr, pstr); // 含'\0'
        } else {
            _pstr = new char[1];
            *_pstr = '\0'; // 空字符串
        }
    }
    ~string() {
        delete[] _pstr;
        _pstr = nullptr;
    }
    string (const string& str) {
        _pstr = new char[strlen(str._pstr) + 1];
        strcpy(_pstr, str._pstr);
    }
    string& operator=(const string& str) {
        if (this == &str) {
            return *this;
        }
        delete[] _pstr;
        _pstr = new char[strlen(str._pstr) + 1];
        strcpy(_pstr, str._pstr);
        return *this;
    }
    bool operator>(const string& str) const { return strcmp(_pstr, str._pstr) > 0; }
    bool operator<(const string& str) const { return strcmp(_pstr, str._pstr) < 0; }
    bool operator==(const string& str) const { return strcmp(_pstr, str._pstr) == 0; }
    int length() const { return strlen(_pstr); }
    char& operator[](const int index) { return _pstr[index]; } // 外部可修改
    const char& operator[](const int index) const { return _pstr[index]; } // 常方法版本 常对象调用
    const char* c_str() const { return _pstr; }
    friend std::ostream& operator<<(std::ostream& out, const string& str);
    friend string operator+(const string& lhs, const string& rhs); // + 重载
private:
    char* _pstr;
};

std::ostream& operator<<(std::ostream& out, const string& str) {
    out << str._pstr;
    return out;
}
string operator+(const string& lhs, const string& rhs) {
    char* ptmp = new char[strlen(lhs._pstr) + strlen(rhs._pstr) + 1]; // 分配足够的空间
    strcpy(ptmp, lhs._pstr);
    strcat(ptmp, rhs._pstr); // 含'\0'
    // return string(ptmp); // 不能直接返回 ptmp 指向的内存泄露了！
    string stmp(ptmp);
    // 这样写是正确的 但是在构造过程中又分配了一次相同的空间
    delete[] ptmp; // 回收了内存
    return stmp;   // 函数结束后 stmp 又被析构
    // 要返回stmp就一定会产生临时对象
    // 临时对象的构造和析构又会进行大量的内存操作
    // 效率非常低
    // 后面来解决这个问题
}

int main() {
    string str1; // 默认构造
    string str2 = "aaa"; // = 重载
    string str3 = "bbb";
    string str4 = str2 + str3; // + 重载
    string str5 = str2 + "ccc";
    string str6 = "ddd" + str2; // 全局 + 重载

    std::cout << "str6: " << str6 << std::endl; // << 重载

    if (str5 > str6) { // > 重载
        std::cout << str5 << " > " << str6 << std::endl;
    } else {
        std::cout << str5 << " < " << str6 << std::endl;
    }

    int len = str6.length(); // length()
    for (int i = 0; i < len; ++ i) {
        std::cout << str6[i] << " "; // [] 重载
    }
    std::cout << std::endl;

    char buf[1024] = {0};
    strcpy(buf, str6.c_str()); // c_str() string -> const char*
    
    return 0;
}
```


## string 的迭代器 iterator

### 容器的迭代器 iterator
迭代器示例：  
```C++
#include <iostream>
#include <string>
int main() {
    std::string str1 = "hello";
    std::string::iterator it = str1.begin(); // iterator 是 string 的内部类
    // begin() 返回的是容器的首元素
    while (it != str1.end()) { // end() 是容器最后元素的后继
        std::cout << *it; // 解引用访问容器底层数据
        ++ it; // 跳到下一个位置
    }
    return 0;
}
```
通过上面的示例可以看到迭代器的特性：  
- 迭代器可以**透明地**遍历容器内部的元素；
- 每一种容器都有自己的数据结构，要用对应的迭代器，所以将迭代器设计为内部类型；
- 都有`begin() end()`方法，分别返回容器首元素和末尾的后继位置的迭代器；
- `++`重载，将迭代器指向后一个位置；
- `*`重载，访问迭代器指向的元素的值。

注意，不同容器的迭代器是不可以进行比较运算的。

### 实现迭代器
```C++
#include <iostream>
#include <cstring>
class string {
public:
    string(const char* pstr = nullptr) {
        if (pstr != nullptr) {
            _pstr = new char[strlen(pstr) + 1];
            strcpy(_pstr, pstr);
        } else {
            _pstr = new char[1];
            *_pstr = '\0';
        }
    }
    ~string() {
        delete[] _pstr;
        _pstr = nullptr;
    }
    string (const string& str) {
        _pstr = new char[strlen(str._pstr) + 1];
        strcpy(_pstr, str._pstr);
    }
    string& operator=(const string& str) {
        if (this == &str) {
            return *this;
        }
        delete[] _pstr;
        _pstr = new char[strlen(str._pstr) + 1];
        strcpy(_pstr, str._pstr);
        return *this;
    }
    bool operator>(const string& str) const { return strcmp(_pstr, str._pstr) > 0; }
    bool operator<(const string& str) const { return strcmp(_pstr, str._pstr) < 0; }
    bool operator==(const string& str) const { return strcmp(_pstr, str._pstr) == 0; }
    int length() const { return strlen(_pstr); }
    char& operator[](const int index) { return _pstr[index]; }
    const char& operator[](const int index) const { return _pstr[index]; }
    const char* c_str() const { return _pstr; }
    friend std::ostream& operator<<(std::ostream& out, const string& str);
    friend string operator+(const string& lhs, const string& rhs);
    // 给string类提供迭代器的实现
    class iterator {
    public:
        iterator(char* p = nullptr) : _p(p) {}
        bool operator!=(const iterator& it) const { return _p != it._p; } // 迭代器不相等就是底层指针不相等
        void operator++() { ++ _p; } // 前置++
        char& operator*() { return *_p; } // 迭代器解引用
    private:
        char* _p;
    };
    iterator begin() { // 返回指向容器首元素的迭代器
        return iterator(_pstr); // 构造迭代器 指向首元素
    }
    iterator end() { // 返回指向容器末尾元素后继的迭代器
        return iterator(_pstr + length());
    }
private:
    char* _pstr;
};
std::ostream& operator<<(std::ostream& out, const string& str) {
    out << str._pstr;
    return out;
}
string operator+(const string& lhs, const string& rhs) {
    char* ptmp = new char[strlen(lhs._pstr) + strlen(rhs._pstr) + 1];
    strcpy(ptmp, lhs._pstr);
    strcat(ptmp, rhs._pstr);
    string stmp(ptmp);
    delete[] ptmp;
    return stmp;
}
int main() {

    string str1 = "hello";
    // string::iterator it = str1.begin();
    auto it = str1.begin();
    while (it != str1.end()) {
        std::cout << *it;
        ++ it;
    }
    std::cout << std::endl;
    // c++11 foreach 遍历容器的元素 底层通过迭代器遍历
    for (char ch : str1) { // 依赖于 begin() 和 end()
        std::cout << ch;
    }
    std::cout << std::endl;
    return 0;
}
```


## vector 的迭代器 iterator
```C++
// ... public
class iterator { // vector的迭代器实现
public: // 经典四方法
    iterator(T* p = nullptr) : _p(p) {}
    bool operator!=(const iterator& it) const { return _p != it._p; }
    void operator++() { ++ _p; }
    T& operator*() { return *_p; }
private:
    T* _p;
};
iterator begin() { return iterator(_first); }
iterator end() { return iterator(_last); }
// ...
```


## 容器迭代器的失效问题

### 什么是迭代器的失效
```C++
#include <iostream>
#include <vector>
using namespace std;
int main() {
	vector<int> vec = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
    for (auto it = vec.begin(); it != vec.end(); ++ it) {
        if (*it > 4) {
            vec.erase(it); // 迭代器失效
        }
    }
    for (auto i : vec) {
        cout << i << " ";
    }
    cout << endl;
	return 0;
}
```
编译运行上述代码，运行时会出错。原因是，在`vec.erase(it)`之后，`erase`将`it`指向的元素删除了，同时，也导致了从`it`指向的位置开始到`vec.end()`的所有迭代器全部失效了，再次对`it`进行运算是非法的。

使用`vec.insert(it, val)`也会出现相同的问题，在`it`位置插入元素后，从`it`指向位置到`vec.end()`的所有迭代器全部失效了，不可以再次对它们进行运算。

同样，当`vector`扩容后，所有指向原来的`vector`的迭代器也会全部失效。

这就是**迭代器的失效问题**。

总结：  
1. `erase`，删除元素，导致当前位置到末尾位置所有迭代器失效；
2. `insert`，插入元素，导致当前位置到末尾位置所有迭代器失效；
3. 导致容器扩容的操作，原来容器的所有迭代器全部失效。

### 如何解决迭代器失效
解决方法：对插入（删除）点的迭代器进行更新。

示例：  
```C++
#include <iostream>
#include <vector>
using namespace std;
int main() {
	vector<int> vec = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
    for (auto it = vec.begin(); it != vec.end(); ) { // 不可以 ++ it
        if (*it > 4) {
            it = vec.erase(it);
            // 删除操作导致 指向当前位置的 it 失效
            // erase 会返回一个指向当前位置的合法的迭代器
            // 用返回的迭代器更新 it 即可
        } else {
            ++ it; // 不删除的情况下 迭代器向后移动
        }
    }
    for (auto i : vec) {
        cout << i << " ";
    }
    cout << endl;
	return 0;
}
```
可以正确运行。

### 迭代器失效原理（vector实现）
在之前写的`vector`的基础上添加迭代器失效的功能，写了一个简单的实现。

实现的原理是：  
1. 在迭代器的成员中添加一个指向容器对象的指针，如果该指针为`nullptr`表明迭代器失效；
2. 在容器的成员中添加一个管理容器对象的全部迭代器的链表，生成指向该容器对象的迭代器时（构造时），将其挂入该链表中；
3. 在容器的成员中添加一个`verify(first, last)`私有成员方法，每当操作导致迭代器失效时（下面完善了`pop_back()`方法），调用该方法清理失效的迭代器；
4. 对迭代器进行运算时，需要检查迭代器的有效性；

```C++
#include <iostream>
#include "allocator.hpp"

template<typename T, typename Alloc = allocator<T>>
class vector {
public:
    vector(int size = 10) {
        _first = _allocator.allocate(size);
        _last = _first;
        _end = _first + size;
    }
    ~vector() {
        for (T* p = _first; p != _last; ++ p) {
            _allocator.destroy(p);
        }
        _allocator.deallocate(_first);
        _first = _last = _end = nullptr;
    }
    vector(const vector<T>& rhs) {
        int size = rhs._end - rhs._first;
        _first = _allocator.allocate(size);
        int len = rhs._last - rhs._first;
        for (int i = 0; i < len; ++ i) {
            _allocator.construct(_first + i, rhs._first[i]);
        }
        _last = _first + len;
        _end = _first + size;
    }
    vector<T>& operator=(const vector<T>& rhs) {
        if (this == &rhs) {
            return *this;
        }
        for (T* p = _first; p != _last; ++ p) {
            _allocator.destroy(p);
        }
        _allocator.deallocate(_first);
        int size = rhs._end - rhs._first;
        _first = _allocator.allocate(size);
        int len = rhs._last - rhs._first;
        for (int i = 0; i < len; ++ i) {
            _allocator.construct(_first + i, rhs._first[i]);
        }
        _last = _first + len;
        _end = _first + size;
        return *this;
    }
    bool full() const { return _last == _end; }
    bool empty() const { return _first == _last; }
    int size() const { return _last - _first; }
    void push_back(const T& val) {
        if (full()) {
            expand();
        }
        _allocator.construct(_last, val);
        ++ _last;
    }
    void pop_back() { // 简单修改一下
        if (!empty()) {
            verify(_last - 1, _last); // 将最后一个元素弹出 则两个最后的位置上的迭代器失效
            -- _last;
            _allocator.destroy(_last);
        }
    }
    T back() const {
        return *(_last - 1);
    }
    T& operator[](int index) { return _first[index]; }
private:
    void expand() {
        int size = _end - _first;
        T* ptmp = _allocator.allocate(size * 2);
        for (int i = 0; i < size; ++ i) {
            _allocator.construct(ptmp + i, _first[i]);
        }
        for (T* p = _first; p != _last; ++ p) {
            _allocator.destroy(p);
        }
        _allocator.deallocate(_first);
        _first = ptmp;
        _last = _first + size;
        _end = _first + size * 2;
    }
public:
    class iterator {
        // 允许 vector<T, Alloc> 访问 iterator 的私有成员
        friend class vector<T, Alloc>;
    public:
        // iterator(T* p = nullptr) : _p(p) {}
        iterator(vector<T, Alloc>* pvec = nullptr, T* p = nullptr)
            : _pvec(pvec) // pvec 表明指向的容器对象
            , _p(p)
        {
            // 创建一个 iterator_base 节点 并头插法插到 容器对象的迭代器链表中
            iterator_base* itb = new iterator_base(this, _pvec->_head._next);
            _pvec->_head._next = itb;
        }
        // bool operator!=(const iterator& it) const { return _p != it._p; }
        bool operator!=(const iterator& it) const {
            // 检测迭代器的有效性
            // 当前迭代器指向的容器对象指针为空 或 容器对象指针和 it 不同
            // 说明失效了
            if (_pvec == nullptr || _pvec != it._pvec) {
                throw "iterator incompatable!";
            }
            return _p != it._p;
        }
        // void operator++() { ++ _p; }
        void operator++() {
            // 检测迭代器的有效性
            if (_pvec == nullptr) {
                throw "iterator invalid!";
            }
            ++ _p;
        }
        // T& operator*() { return *_p; }
        T& operator*() {
            // 检测迭代器的有效性
            if (_pvec == nullptr) {
                throw "iterator invalid!";
            }
            return *_p;
        }
    private:
        T* _p;
        vector<T, Alloc>* _pvec; // 当前的迭代器是哪个容器对象的迭代器
    };
    // iterator begin() { return iterator(_first); }
    // iterator end() { return iterator(_last); }
    iterator begin() { return iterator(this, _first); } // 构造函数更新了
    iterator end() { return iterator(this, _last); }
    
private:
    // 管理容器对象的所有迭代器的链表
    struct iterator_base {
        iterator_base(iterator* c = nullptr, iterator_base* n = nullptr) : _cur(c), _next(n) {}
        iterator* _cur;
        iterator_base* _next;
    };
    // 链表头
    iterator_base _head; // 链表的释放还没写
    // 检测迭代器是否失效
    void verify(T* first, T* last) {
        // 遍历指向该容器对象的所有迭代器
        // 让在 first 和 last 之间的迭代器失效
        iterator_base* pre = &this->_head;
        iterator_base* it = this->_head._next;
        while (it != nullptr) {
            if (it->_cur->_p >= first && it->_cur->_p <= last) {
                it->_cur->_pvec = nullptr; // 迭代器失效 指向空对象
                pre->_next = it->_next; // 从链表上删除
                delete it;
                it = pre->_next;
            } else {
                pre = it;
                it = it->_next;
            }
        }
    }
private:
    T* _first;
    T* _last;
    T* _end;
    Alloc _allocator;
};

int main() {
    vector<int> vec;
    for (int i = 0; i < 10; ++ i) {
        vec.push_back(rand() % 100 + 1);
    }
    auto it1 = vec.end();
    vec.pop_back(); // verify(_last-1, _last);
    vector<int>::iterator it2 = vec.end();
    std::cout << (it1 != it2) << std::endl;
    return 0;
}
```


## 深入理解 new 和 delete 的原理

### new delete 和 malloc free
回顾一下它们的区别，`new  delete`：  
1. `malloc`按字节分配内存空间，返回值为`void*`；`new`分配时需要指定类型，返回指定类型的指针；
2. `malloc`只负责分配内存空间；`new`不仅分配内存空间，同时进行数据的初始化；
3. `malloc`分配内存失败返回`nullptr`空指针；`new`分配失败时会抛出`bad_alloc`类型的异常。

`free  delete`：  
1. `free`仅释放内存；`delete`释放内存时也调用析构函数。

### new operator 和 operator new

当我们使用类似这样的语句：`Test t = new Test;`，这里的`new`是C++提供的一个操作符，它**总会**做两件事情：  
1. 为指定类型的对象分配内存；
2. 调用对象的构造函数初始化内存中的对象（如果是一个对象）。

`new operator`（操作符）通过调用一个函数来分配内存，这个函数就是全局的`::operator new`。它通常这样定义：  
```C++
void* operator new(size_t size) {
    void* p = malloc(size);
    if (p == nullptr) {
        throw bad_alloc(); // 内存分配失败会抛出异常
    }
    return p;
}
```

当然我们也可以提供我们重载的版本，此时就不会使用C++提供的版本，例如：  
```C++
#include <iostream>
void* operator new(size_t size) {
    std::cout << "my operator new" << std::endl;
    void* p = malloc(size);
    if (p == nullptr) {
        throw std::bad_alloc();
    }
    return p;
}
void operator delete(void* ptr) {
    std::cout << "my operator delete" << std::endl;
    free(ptr);
}
class Test {
public:
    Test() { std::cout << "Test()" << std::endl; }
    ~Test() { std::cout << "~Test()" << std::endl; }
};
int main() {
    Test* t = new Test;
    delete t;
    return 0;
}
/* 输出
xushun@xushun-virtual-machine:~/cppmiddle$ ./a.out 
my operator new
Test()
~Test()
my operator delete
*/
```

### delete operator 和 operator delete

`delete`的情况和`new`类似，`delete`同样是C++提供的操作符，它也总会做两件事情：  
1. 调用对象的析构函数（如果是一个对象）；
2. 释放对象使用的内存。

`delete operator`（操作符）通过调用一个函数来释放内存，这个函数就是全局的`::operator delete`。它通常这样定义：  
```C++
void operator delete(void* ptr) {
    free(ptr);
}
```

### operator new[] 和 operator delete[]
类似的，`new[] delete[]`操作符依赖`operator new[]`和`operator delete[]`来分配内存：  
```C++
void* operator new[](size_t size) {
    void* p = malloc(size);
    if (p == nullptr) {
        throw std::bad_alloc();
    }
    return p;
}
void operator delete[](void* ptr) {
    free(ptr);
}
```


### 为什么 new delete 和 new[] delete[] 不能混用

测试时，可以使用`valgrind`工具来检测内存泄漏。

> `valgrind --leak-check=full ./a.out`

#### 内置类型
对于编译器内置的类型，如`int double`，**可以**混用`new delete[]`和`new[] delete`。

原因是，编译器内置类型没有构造和析构的工作，而在内存分配时，`operator new`和`operator new[]`，`operator delete`和`operator delete[]`并没有区别。所以分配和释放内存时，都不会出错。

但不推荐这样的写法。

#### 带有析构函数的类类型
对于带有析构函数的类类型（编译器默认提供的析构不做任何事情，不算），为了能够正确的进行析构，在`Test* p = new Test[100]`的时候，会在内存块起始位置（`[p] - 0x08`）多开辟8个字节，用于记录对象的个数以便析构；而在`delete[] p`时，会从`[p] - 0x08`位置读取对象的个数，然后对每个对象（`p + i * sizeof(Test)`）进行析构，析构完成后再`free(p - 0x08)`从起始位置释放内存空间。

> 对于编译器内置类型和不带析构函数的类类型，`new[]`分配内存时，不会多分配8字节以记录对象数量，那么`delete[]`时如何确定对象数量呢？  
> 我猜，是系统调用做的。`free`时，只要传入起始地址，系统内核就会将之前分配的一块内存释放掉。而如果传入的地址不是内核之前分配的某一块内存的起始地址，就会报错。

测试：  
```C++
#include <iostream>
void* operator new(size_t size) {
    void* p = malloc(size);
    if (p == nullptr) {
        throw std::bad_alloc();
    }
    std::cout << "operator new:      " << p << std::endl;
    return p;
}
void operator delete(void* ptr) {
    std::cout << "operator delete:   " << ptr << std::endl;
    free(ptr);
}
void* operator new[](size_t size) {
    void* p = malloc(size);
    if (p == nullptr) {
        throw std::bad_alloc();
    }
    std::cout << "operator new[]:    " << p << std::endl;
    return p;
}
void operator delete[](void* ptr) {
    std::cout << "operator delete[]: " << ptr << std::endl;
    free(ptr);
}

class Test {
public:
    Test() { std::cout << "Test()" << std::endl; }
    ~Test() { std::cout << "~Test()" << std::endl; }
private:
    int a;
};

int main() {
    int* p1 = new int[5];
    std::cout << "access addr: " << p1 << std::endl;
    delete[] p1;
    std::cout << "------------------------------------------" << std::endl;

    int* p2 = new int[5];
    std::cout << "access addr: " << p2 << std::endl;
    delete p2;
    std::cout << "------------------------------------------" << std::endl;

    int* p3 = new int;
    std::cout << "access addr: " << p3 << std::endl;
    delete[] p3;
    std::cout << "------------------------------------------" << std::endl;

    int* p4 = new int;
    std::cout << "access addr: " << p4 << std::endl;
    delete p4;
    std::cout << "------------------------------------------" << std::endl;

    Test* t1 = new Test[5];
    std::cout << "access addr: " << t1 << std::endl;
    delete[] t1;
    std::cout << "------------------------------------------" << std::endl;

    Test* t2 = new Test[5];
    std::cout << "access addr: " << t2 << std::endl;
    delete t2;
    std::cout << "------------------------------------------" << std::endl;

    Test* t3 = new Test;
    std::cout << "access addr: " << t3 << std::endl;
    delete t3;
    std::cout << "------------------------------------------" << std::endl;

    Test* t4 = new Test;
    std::cout << "access addr: " << t4 << std::endl;
    delete[] t4;
    std::cout << "------------------------------------------" << std::endl;

    return 0;
}
```




## new delete 重载实现的对象池应用




