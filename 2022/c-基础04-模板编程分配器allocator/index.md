# 【C++基础】04 - 模板编程、分配器allocator


## 函数模板
相关的概念：  
- 函数模板：不进行编译，因为调用前不知道类型
- 模板的实例化：函数调用点进行实例化
- 模板函数：需要编译的代码
- 模板类型参数：`typename/class`修饰，逗号分隔
- 模板非类型参数：常量，只能使用，不能修改，必须是整型类型（包括地址和引用）
- 模板的实参推演：根据用户传入的实参类型，推导模板类型参数的类型
- 模板的特化（专用化）：特殊的实例化（不是编译器默认生成的，而是用户提供的）
- 函数模板、模板特化、非模板函数的重载关系：严格意义上不是重载关系，而是共存，因为它们的函数名是不同的

函数模板的意义：类型也可以进行参数化，只需写一套函数模板，就可以适用于多种类型参数。

示例代码：  
```C++
#include <cstring>
#include <iostream>
using namespace std;
template<typename T> // 定义一个模板参数列表 typename也可以是class 多个参数逗号隔开
bool compare(T a, T b) { // compare现在只是一个函数模板
    cout << "template compare" << endl;
    return a > b;
}
// const char*类型的特化版本
template<> // template不能省略 但是模板参数列表应为空
bool compare<const char*>(const char* a, const char* b) {
    cout << "specific compare" << endl;
    return strcmp(a, b) > 0;
}
// 普通函数 非模板函数
bool compare(const char* a, const char*b) {
    cout << "normal compare" << endl;
    return strcmp(a, b) > 0;
}
int main() {
    // 函数调用点
    compare<int>(10, 20); // 使用compare时 需要指定一个类型参数
                          // 这里指定了int类型
                          // 模板名 + 参数列表 = 函数
    compare(20, 30); // 也可以这样调用
                     // 可以根据用户传入的实参类型 来推导出模板类型参数的具体类型
    // compare(20, 30.5); // 错误写法
                          // 未指定模板参数类型 依赖模板的实参推演
                          // 但是模板参数列表中只有一种类型 编译器不知道应该使用int还是double
    compare<int>(20, 30.5); // 30.5被转换为int类型
    compare("aaa", "bbb"); // 实参推演 -> const char* 类型
                           // 比较const char*的大小 相当于比较地址大小 无意义
                           // 需要模板特化
                           // 如果有compare的普通函数（非模板函数） 就会直接使用普通函数
    compare<const char*>("aaa", "bbb"); // 这样会调用特化的模板函数
    return 0;
/* 输出
    template compare
    template compare
    template compare
    normal compare
    specific compare
*/
}
```

当程序运行到**函数调用点**，调用的函数由**函数模板**定义时，编译器就会根据用户指定的类型，从原函数模板中实例化一份函数代码出来，例如`compare<int>(10, 20>;`时，**函数模板的实例化**：`bool compare<int>(int a, int b) { return a > b; }`。实例化得到的函数也称**模板函数**。指定不同的类型时，编译器都会根据具体类型实例化出一份对应的模板函数代码。

每个类型对应的实例化代码，只有一份。如果再次调用`compare<int>(1, 2);`，则不需要再次实例化，直接调用之前生成的模板函数即可。

从编译器的角度来说，函数模板是无法编译的，当每实例化一种类型的模板函数，才需要对其进行编译；  
从用户的角度来说，用户需要编写的代码确实减少了，对于同一逻辑的不同类型，只需要编写一份代码即可。

使用模板函数时，也可以不指定具体类型，如`compare(20, 30);`，编译器会根据你传入的实参，推导出模板类型参数的具体类型。称作**模板的实参推演**。

当程序运行到这一句时，`compare("aaa", "bbb");`，编译器进行实参推演，得到`const char*`类型，实例化为`bool compare<const char*>(const char* a, const char* b) { return a > b; }`，实际上是在比较字符串地址的大小，没有意义。这说明，对于某些类型来说，依赖编译器默认实例化的模板函数，逻辑并不正确。这是就需要用户手动编写逻辑正确的版本，这也被称为**函数模板的特化**。这样一来，对于`const char*`类型的模板函数，就会直接调用我们给定的特化版本，而无需实例化。


### 函数模板的使用
模板的代码，不可以在一个源文件（.cpp）中定义，在另一个源文件中使用。如果要在某个源文件中使用模板，必须要在该文件中看到模板的定义，这样才能进行正常的实例化，产生可以编译的代码。

所以，模板代码应该放在头文件中，然后在要使用的源文件中`#include`包含，预编译时直接展开定义。

为什么不能在另一个源文件中定义模板呢？分析如下：  
首先考虑下面的两个源文件：  
```C++
// main.cpp
template<typename T>
bool compare(T a, T b);
int main() {
    compare<int>(10, 20);
    compare(20.5, 30.5);
    compare("aaa", "bbb");
    compare<const char*>("aaa", "bbb");
    return 0;
}

// test.cpp
#include <cstring>
#include <iostream>
using namespace std;
template<typename T>
bool compare(T a, T b) {
    cout << "template compare" << endl;
    return a > b;
}
template<>
bool compare<const char*>(const char* a, const char* b) {
    cout << "specific compare" << endl;
    return strcmp(a, b) > 0;
}
bool compare(const char* a, const char*b) {
    cout << "normal compare" << endl;
    return strcmp(a, b) > 0;
}
```
在`test.cpp`中，定义了`compare`函数模板，并且提供了一个特化版本`compare<const char*>`，以及一个普通`compare`函数。

`g++ -c main.cpp test.cpp`，生成可重定位目标文件，可以成功生成（`main.o test.o`）。  
`g++ main.o test.o`，链接出错：  
```console
xushun@xushun-virtual-machine:~/cppmiddle/test$ g++ -c main.cpp test.cpp
xushun@xushun-virtual-machine:~/cppmiddle/test$ g++ main.o test.o
/usr/bin/ld: main.o: in function `main':
main.cpp:(.text+0x13): undefined reference to `bool compare<int>(int, int)'
/usr/bin/ld: main.cpp:(.text+0x28): undefined reference to `bool compare<double>(double, double)'
collect2: error: ld returned 1 exit status
```
从错误中可知，链接时`compare<int> compare<double>`找不到定义，这是为什么呢？

答：  
- 在编译`main.cpp`时，只能找到`compare`函数模板的声明，找不到定义，所以执行到函数调用点时，编译器无法根据函数模板的定义来实例化模板函数并编译，所以在符号表中`compare<int>  compare<double>`都是`*UND*`；
- 编译`test.cpp`时，不会编译函数模板，同时没有特化`compare<int>  compare<double>`，也就不会编译这两个模板函数（不会生成它们的定义）；
- 当链接时，链接器进行符号解析，在`test.o`中无法找到`comapre<int>  compare<double>`的定义（符号表中没有它们的符号），所以出现链接错误；
- 编译`main.cpp`时，`compare<const char*>`同样无法实例化，但是在`test.cpp`中存在它的特化版本（会被编译），所以链接时可以找到它的定义；
- 因为`main.cpp`中只有`compare`函数模板的声明，所以`compare("aaa", "bbb");`和`compare<const char*>("aaa", "bbb");`都编译为`compare<const char*>("aaa", "bbb");`，如果想编译为普通函数，需要将普通函数在`main.cpp`中声明。



## 类模板

示例代码：  
```C++
#include <iostream>
using namespace std;

template<typename T> // 模板参数列表
class SeqStack { // 类模板
public:
    // 构造和析构函数名不用加 <T>
    // 其他地方出现该类 需要加上参数列表<T>
    SeqStack(int size = 10)
        : _pstack(new T[size])
        , _top(0)
        , _size(size)
    {}
    ~SeqStack() {
        delete[] _pstack;
        _pstack = nullptr;
    }
    SeqStack(const SeqStack<T>& stack) // 拷贝构造
        : _top(stack._top)
        , _size(stack._size)
    {
        _pstack = new T[_size];
        for (int i = 0; i < _top; ++ i) {
            _pstack[i] = stack._pstack[i];
        }
    }
    SeqStack<T>& operator=(const SeqStack<T>& stack) { // 赋值重载
        if (this == &stack) {
            return *this;
        }
        delete[] _pstack;
        _top = stack._top;
        _size = stack._size;
        _pstack = new T[_size];
        for (int i = 0; i < _top; ++ i) {
            _pstack[i] = stack._pstack[i];
        }
        return *this;
    }
    void push(const T& val) {
        if (full()) {
            resize();
        }
        _pstack[_top ++] = val;
    }
    void pop() {
        if (empty()) {
            return;
        }
        -- _top;
    }
    T top() const {
        if (empty()) {
            throw "stack is empty"; // 抛出异常也代表函数逻辑结束
        }
        return _pstack[_top - 1];
    }
    bool full() const {
        return _top == _size;
    }
    bool empty() const; // 声明
private:
    void resize() {
        T* ptmp = new T[_size * 2];
        for (int i = 0; i < _top; ++ i) {
            ptmp[i] = _pstack[i];
        }
        delete[] _pstack;
        _pstack = ptmp;
        _size *= 2;
    }
private:
    T* _pstack;
    int _top;
    int _size;
};
// 在类外定义成员方法
template<typename T> // 模板参数列表作用范围只到离它最近的类体/函数体内
bool SeqStack<T>::empty() const {
    return _top == 0;
}

int main() {
    // SeqStack<int> 类模板实例化为一个模板类 class SeqStack<int>{};
    SeqStack<int> s1;  // 实例化一个模板类的对象
                       // 这里仅实例化了 构造和析构函数 因为其他的成员方法还没有被调用
    s1.push(10); // 实例化 SeqStack<int>::push(const SeqStack<int>& stack)
    s1.push(20);
    s1.push(30);
    s1.push(40);
    s1.push(50);
    s1.pop(); // 实例化 SeqStack<int>::push()
    cout << s1.top() << endl; // 实例化 SeqStack<int>::top()
    // 没有调用的成员方法 不会被实例化 编译
    return 0;
}
```

类模板的选择性实例化：类模板被实例化为一个模板类并实例化了对象时，仅仅对模板类的构造和析构进行实例化。只有在其他成员方法被调用时，才会对其进行实例化。


## 练习 - 实现vector

```C++
#ifndef __VECTOR_HPP_
#define __VECTOR_HPP_

template<typename T>
class vector {
public:
    vector(int size = 10) {
        _first = new T[size];
        _last = _first;
        _end = _first + size;
    }
    ~vector() {
        delete[] _first;
        _first = _last = _end = nullptr;
    }
    vector(const vector<T>& rhs) {
        int size = rhs._end - rhs._first;
        _first = new T[size];
        int len = rhs._last - rhs._first;
        for (int i = 0; i < len; ++ i) {
            _first[i] = rhs._first[i];
        }
        _last = _first + len;
        _end = _first + size;
    }
    vector<T>& operator=(const vector<T>& rhs) {
        if (this == &rhs) {
            return *this;
        }
        delete[] _first;
        int size = rhs._end - rhs._first;
        _first = new T[size];
        int len = rhs._last - rhs._first;
        for (int i = 0; i < len; ++ i) {
            _first[i] = rhs._first[i];
        }
        _last = _first + len;
        _end = _first + size;
        return *this;
    }
    bool full() const { return _last == _end; } // 数组满
    bool empty() const { return _first == _last; } // 数组空
    int size() const { return _last - _first; } // 元素个数
    void push_back(const T& val) { // 向末尾添加元素
        if (full()) {
            expand();
        }
        *_last ++ = val;
    }
    void pop_back() { // 删除末尾元素
        if (!empty()) {
            -- _last;
        }
    }
    T back() const { // 返回末尾元素
        return *(_last - 1);
    }
private:
    void expand() { // 内存扩容
        int size = _end - _first;
        T* ptmp = new T[size * 2];
        for (int i = 0; i < size; ++ i) {
            ptmp[i] = _first[i];
        }
        delete[] _first;
        _first = ptmp;
        _last = _first + size;
        _end = _first + size * 2;
    }
private:
    T* _first;  // 数组起始
    T* _last;   // 有效元素的后继位置
    T* _end;    // 数组空间的后继位置
};

#endif
```



## 理解并实现一个 allocator

上面`vector`的实现非常简单，当容器元素是简单类型时，没有什么问题，但如果容器元素是对象时，在内存分配释放以及对象构造析构方面就存在许多问题。

这就需要`allocator`来进行管理。

### vector存在什么问题？
#### vector构造函数  
```C++
class Test {
public:
    Test() { std::cout << "Test()" << std::endl; }
    ~Test() { std::cout << "~Test()" << std::endl; }
};
int main() {
    vector<Test> vec;
    return 0;
}
/* 输出
xushun@xushun-virtual-machine:~/cppmiddle$ ./alloc 
Test()
Test()
Test()
Test()
Test()
Test()
Test()
Test()
Test()
Test()
~Test()
~Test()
~Test()
~Test()
~Test()
~Test()
~Test()
~Test()
~Test()
~Test()
*/
```
在上面的代码中，我们仅仅定义了一个`Test`类型的`vector`，但是调用了10次`Test`的构造和析构函数。这是因为`vector`的构造函数中，使用了`new`，`new`不仅会分配堆内存，也会调用类的构造函数。

显然，这样的操作是不合理的。在`vector`的构造函数中，我们只需要为容器分配堆内存空间，而并不需要给每个对象调用构造函数，因为此时还未向容器中添加元素。

我们需要把**分配内存**和**对象构造**分离开。

#### vector析构函数
再说`vector`的析构函数，在析构函数中，使用了`delete[] _first;`，`delete[]`操作会将数组中的所有对象析构，并且释放堆内存。

这显然也是错误的，因为容器数组中，可能只有很少的有效元素，而使用`delete[]`会对所有元素进行析构。可我们只需要对有效元素进行析构即可，然后释放整个数组空间。

我们需要将**释放内存**和**对象析构**分离开。

#### vector::push_back
```C++
class Test {
public:
    Test() { std::cout << "Test()" << std::endl; }
    ~Test() { std::cout << "~Test()" << std::endl; }
    Test(const Test& t) { std::cout << "Test(const Test&)" << std::endl; }
    Test& operator=(const Test& t) { std::cout <<  "operator=(const Test&)" << std::endl; return *this; }
};

int main() {
    Test t1, t2;
    vector<Test> vec(2);
    vec.push_back(t1);
    vec.push_back(t2);
    return 0;
}
/* 输出
xushun@xushun-virtual-machine:~/cppmiddle$ ./alloc 
Test()
Test()
Test()
Test()
operator=(const Test&)
operator=(const Test&)
~Test()
~Test()
~Test()
~Test()
*/
```
我们使用`push_back(t1)`的想法是，在容器数组中，构造一个和`t1`相同的新对象（拷贝构造），而我们实现的`vector`在构造函数中已经构造过了，只能用`t1`给某个已构造的对象赋值（赋值重载）。

我们需要在添加元素时进行对象的构造，而不是提前构造好进行赋值。

#### vector::pop_back
```C++
class Test {
public:
    Test() { std::cout << "Test()" << std::endl; }
    ~Test() { std::cout << "~Test()" << std::endl; }
    Test(const Test& t) { std::cout << "Test(const Test&)" << std::endl; }
    Test& operator=(const Test& t) { std::cout <<  "operator=(const Test&)" << std::endl; return *this; }
};
int main() {
    Test t1, t2;
    std::cout << "-------------------------" << std::endl;
    vector<Test> vec(2);
    std::cout << "-------------------------" << std::endl;
    vec.push_back(t1);
    vec.push_back(t2);
    std::cout << "-------------------------" << std::endl;
    vec.pop_back();
    std::cout << "-------------------------" << std::endl;
    return 0;
}
/* 输出
xushun@xushun-virtual-machine:~/cppmiddle$ ./alloc 
Test()
Test()
-------------------------
Test()
Test()
-------------------------
operator=(const Test&)
operator=(const Test&)
-------------------------
-------------------------
~Test()
~Test()
~Test()
~Test()
*/
```
`pop_back`的问题更加显而易见，`pop_back()`时，仅仅只是将`_last`指针前移，而没有调用删除对象的析构函数，这样可能导致内存泄漏或者其他问题。

我们需要在删除元素时，进行析构。

### 实现空间配置器allocator
根据上面的分析，我们的`allocator`需要做四件事情：  
1. 内存分配
2. 内存释放
3. 对象构造
4. 对象析构

实现：  
```C++
template<typename T>
class allocator {
public:
    T* allocate(size_t size) { // 内存分配
        return (T*)malloc(sizeof(T) * size);
    }
    void deallocate(void* p) { // 内存释放
        free(p);
    }
    void construct(T* p, const T& val) { // 对象构造 在指定地址p处构造一个和val相同的对象
        new (p) T(val); // 定位new 拷贝构造
    }
    void destroy(T* p) { // 对象析构 p指向的对象进行析构
        p->~T(); // 手动析构
    }
};

template<typename T, typename Alloc = allocator<T>> // 更新模板参数列表
class vector {
public:
    vector(int size = 10) { // 更新构造函数允许用户指定配置器
        // _first = new T[size];
        _first = _allocator.allocate(size); // 仅分配内存
        _last = _first;
        _end = _first + size;
    }
    ~vector() {
        // delete[] _first;
        for (T* p = _first; p != _last; ++ p) {
            _allocator.destroy(p); // 析构有效元素
        }
        _allocator.deallocate(_first); // 释放整个数组内存
        _first = _last = _end = nullptr;
    }
    vector(const vector<T>& rhs) {
        int size = rhs._end - rhs._first;
        // _first = new T[size];
        _first = _allocator.allocate(size); // 仅分配内存
        int len = rhs._last - rhs._first;
        for (int i = 0; i < len; ++ i) {
            // _first[i] = rhs._first[i]; // 对象还未构造 自然无法赋值
            _allocator.construct(_first + i, rhs._first[i]); // 在分配好的内存上构造对象
        }
        _last = _first + len;
        _end = _first + size;
    }
    vector<T>& operator=(const vector<T>& rhs) {
        if (this == &rhs) {
            return *this;
        }
        // delete[] _first;
        for (T* p = _first; p != _last; ++ p) {
            _allocator.destroy(p); // 析构有效元素
        }
        _allocator.deallocate(_first); // 释放整个数组内存
        int size = rhs._end - rhs._first;
        // _first = new T[size];
        _first = _allocator.allocate(size); // 仅分配内存
        int len = rhs._last - rhs._first;
        for (int i = 0; i < len; ++ i) {
            // _first[i] = rhs._first[i]; // 对象还未构造 自然无法赋值
            _allocator.construct(_first + i, rhs._first[i]); // 在分配好的内存上构造对象
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
        // *_last ++ = val;
        _allocator.construct(_last, val);
        ++ _last;
    }
    void pop_back() {
        if (!empty()) {
            -- _last;
            _allocator.destroy(_last); // 析构要删除的元素
        }
    }
    T back() const {
        return *(_last - 1);
    }
private:
    void expand() {
        int size = _end - _first;
        // T* ptmp = new T[size * 2];
        T* ptmp = _allocator.allocate(size * 2); // 仅分配内存
        for (int i = 0; i < size; ++ i) {
            // ptmp[i] = _first[i];
            _allocator.construct(ptmp + i, _first[i]);
        }
        // delete[] _first;
        for (T* p = _first; p != _last; ++ p) {
            _allocator.destroy(p);
        }
        _allocator.deallocate(_first);
        _first = ptmp;
        _last = _first + size;
        _end = _first + size * 2;
    }
private:
    T* _first;
    T* _last;
    T* _end;
    Alloc _allocator; // 使用allocator
};


class Test {
public:
    Test() { std::cout << "Test()" << std::endl; }
    ~Test() { std::cout << "~Test()" << std::endl; }
    Test(const Test& t) { std::cout << "Test(const Test&)" << std::endl; }
    Test& operator=(const Test& t) { std::cout <<  "operator=(const Test&)" << std::endl; return *this; }
};

int main() {
    Test t1, t2;
    std::cout << "-------------------------" << std::endl;
    vector<Test> vec(2);
    std::cout << "-------------------------" << std::endl;
    vec.push_back(t1);
    vec.push_back(t2);
    std::cout << "-------------------------" << std::endl;
    vec.pop_back();
    std::cout << "-------------------------" << std::endl;
    return 0;
}
```

我们再看一下输出：  
```console
xushun@xushun-virtual-machine:~/cppmiddle$ ./alloc 
Test()
Test()
-------------------------
-------------------------
Test(const Test&)
Test(const Test&)
-------------------------
~Test()
-------------------------
~Test()
~Test()
~Test()
```
从上到下依次：  
- 栈上的两个对象构造；
- `vector`初始化时仅分配了内存，没有构造对象；
- `push_back`两个对象，拷贝构造；
- `pop_back`一个对象，进行了析构；
- 容器中剩余的一个对象，和栈上的两个对象，析构。
