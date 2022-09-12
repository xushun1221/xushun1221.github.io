# 【C++高级】03 - 绑定器、函数对象、lambda


绑定器和函数对象都是c++11从boost库中引入的机制。

## bind1st bind2nd 绑定器

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

`function`函数对象类型，可以对函数，或者函数对象，进行包装。

为什么使用`function`类型？  
绑定器、函数对象、lambda表达式，它们都只能使用在一条语句中。如果我们想要将绑定器、函数对象、lambda表达式它们的**类型保留下来**，在多个地方使用，那么就需要用`function`函数对象类型。

### function 使用

小结：  
1. 通过函数类型实例化`function`对象；
2. 通过`function`调用`operator()`时，需要根据函数类型传入相应的参数。

使用示例：  
```C++
#include <iostream>
#include <functional>
using namespace std;

void hello1() { cout << "hello world" << endl; }
void hello2(string str) { cout << str << endl; }
int sum(int a, int b) { return a + b; }
class Test { public: void hello(string str) { cout << str << endl; } };

int main() {
    /* function 类希望用一个函数类型实例化 */
    function<void()> func1 = hello1; /* void() 表示返回值为void 参数列表为空的函数类型 */
    func1(); /* func1.operator()() => hello1() */
    
    function<void(string)> func2 = hello2;
    func2("function<void(string)>");
    
    function<int(int, int)> func3 = sum; /* int(int, int) 表示返回值为int 参数列表为int, int */
    cout << func3(10, 20) << endl;

    /* function 可以保存一个函数对象的类型 */
    function<int(int, int)> func4 = [](int a, int b)->int{ return a + b; };
    cout << func4(100, 200) << endl;

    /* 类的成员方法的参数列表中需要包含对象指针 */
    function<void(Test*, string)> func5 = &Test::hello;
    Test a;
    func5(&a, "Test::hello()"); /* 调用成员方法 必须依赖一个对象 */
    return 0;
}
```

为什么不直接调用函数，而是要用`function`进行包装？看下面的例子：

```C++
#include <iostream>
#include <functional>
#include <map>
using namespace std;

void doHello() { cout << "hello" << endl; }
void doBye() { cout << "bye" << endl; }

int main() {
    int choice = 0;
    map<int, function<void()>> actionMap;
    actionMap[1] = doHello;
    actionMap[2] = doBye;
    for (;;) {
        cout << "1. hello" << endl;
        cout << "2. bye" << endl;
        cout << "choose your action" << endl;
        cin >> choice;
        auto it = actionMap.find(choice);
        if (it == actionMap.end()) {
            cout << "choose again" << endl;
        } else {
            it->second();
        }
        /* 使用下面这种方式 无法做到开-闭原则 */
        /*
            switch (choice) {
            case 1:
                doHello(); break;
            case 2:
                doBye(); break;
            default: break;
            }
        */
    }
    return 0;
}
```



### function 实现原理

#### 模板的完全特例化和部分特例化

**模板的特例化**也叫特殊的实例化，特例化不是编译器做的而是用户提供的，因为编译器对于某些类型提供的实例化逻辑上不正确。如果特例化中提供了确定的类型，就叫做**完全特例化**，否则叫做**部分特例化**。

示例1：  
```C++
#include <iostream>
#include <cstring>
using namespace std;

template <typename T>
bool compare(T a, T b) { return a < b; }

/* 提供对字符串类型的特例化 */
template <>
bool compare<const char*>(const char* a, const char* b) {
    /* 完全特例化  因为它的参数都是确定的 */
    return strcmp(a, b) < 0;
}

int main() {
    cout << compare(10, 20) << endl;     /* 正确的实例化 */
    cout << compare("aa", "bb") << endl; /* 编译器无法提供正确的实例化 需要我们添加特例化 */
    return 0;
}
```

示例2：  
```C++
#include <iostream>
#include <cstring>
using namespace std;

template <typename T>
class Vector {
public:
    Vector() { cout << "Vector" << endl; }
};

/* 完全特例化 */
template <>
class Vector<char*> {
public:
    Vector() { cout << "Vector<char*>" << endl; }
};

/* 针对指针类型提供的特例化版本  但是没有指定具体的指针类型  所以是部分特例化 */
template <typename Ty>
class Vector<Ty*> {
public:
    Vector() { cout << "Vector<Ty*>" << endl; }
};

/* 为返回值为R 有两个参数的A1A2 的函数指针 提供部分特例化 */
/* 将函数返回值和参数也参数化了 */
template<typename R, typename A1, typename A2>
class Vector<R(*)(A1, A2)> { /* 没有提供返回值和参数的具体类型 所以是部分特例化 */
public:
    Vector() { cout << "Vector<R(*)(A1, A2)>" << endl; }
};

int main() {
    Vector<int> vec1;   // 没有特例化版本  编译器默认实例化
    Vector<char*> vec2; // 有完全特例化版本  使用完全特例化
    Vector<int*> vec3;  // 没有完全特例化版本 使用部分特例化
    Vector<int(*)(int, int)> vec4; // 部分特例化
    Vector<int(int, int)> vec5; // 默认特例化
    return 0;
}
/*
xushun@xushun-virtual-machine:~/cppadvanced$ g++ telihua.cpp && ./a.out
Vector
Vector<char*>
Vector<Ty*>
Vector<R(*)(A1, A2)>
Vector
*/
```



#### function 实现原理

`function`使用一个模板类来实现，用一个成员变量保存实例化对象的函数指针，调用`operator()`时调用该函数指针。对于不同参数数量的函数类型，分别对模板类进行不同的特例化。

```C++
#include <iostream>
using namespace std;

template <typename Fty>
class my_function {  };

template <typename R, typename A1>
class my_function<R(A1)> {
    /* 函数类型为(返回值为R 参数列表为A1) 的模板特例化 */
public:
    /* 函数指针类型 */
    using PFUNC = R(*)(A1);
    /* 构造函数  保存传入的函数指针类型 */
    my_function(PFUNC pfunc) : _pfunc(pfunc) {}
    /* operator() 小括号运算符重载时 调用传入的函数 */
    R operator()(A1 arg) { return _pfunc(arg); }
private:
    PFUNC _pfunc;
};

template <typename R, typename A1, typename A2>
class my_function<R(A1, A2)> {
public:
    using PFUNC = R(*)(A1, A2);
    my_function(PFUNC pfunc) : _pfunc(pfunc) {}
    R operator()(A1 arg1, A2 arg2) { return _pfunc(arg1, arg2); }
private:
    PFUNC _pfunc;
};

void hello(string str) { cout << str << endl; }
int sum(int a, int b) { return a + b; }

int main() {
    my_function<void(string)> func1 = hello;
    func1("hello world"); 
    /* func1.operator()("hello world") 底层调用的还是hello这个函数 */
    my_function<int(int, int)> func2 = sum;
    cout << func2(10, 20) << endl;
    return 0;
}
/*
xushun@xushun-virtual-machine:~/cppadvanced$ g++ functiontest3.cpp && ./a.out
hello world
30
*/
```


当然为每种参数个数的函数都写一个特例化，很麻烦，c++还提供了另一种可变参数的写法，只需要写一次即可。

```C++
#include <iostream>
using namespace std;

template <typename Fty>
class my_function {  };

template <typename R, typename...A> /* ... 表示可变参 一组参数 个数任意 */
class my_function<R(A...)> {
public:
    using PFUNC = R(*)(A...);
    my_function(PFUNC pfunc) : _pfunc(pfunc) {}
    /* 这里的 arg 表示一组参数 调用函数时会将所有参数都传入*/
    R operator()(A... arg) { return _pfunc(arg...); }
private:
    PFUNC _pfunc;
};

void hello(string str) { cout << str << endl; }
int sum(int a, int b) { return a + b; }

int main() {
    my_function<void(string)> func1 = hello;
    func1("hello world"); 

    my_function<int(int, int)> func2 = sum;
    cout << func2(10, 20) << endl;
    return 0;
}
```




## c++11 bind 绑定器

c++11 提供了功能更强大的绑定器 `bind`。

它是一个函数模板，可以自动推演模板类型参数。STL中提供的`bind1st`和`bind2nd`只能绑定二元函数对象，而c++11提供的`bind`则没有这个限制。

`bind`可以绑定一个函数或函数对象，它的返回值仍然是一个函数对象。

使用示例：  
```C++
#include <iostream>
#include <functional>
using namespace std;

void hello(string str) { cout << str << endl; }
int sum(int a, int b) { return a + b; }
class Test{ public: int sum(int a, int b) { return a + b; } };

int main() {
    /* bind 返回一个函数对象 使用operaotr()调用函数 */
    bind(hello, "hello bind")();
    cout << bind(sum, 10, 20)() << endl;
    /* 成员方法同样需要依赖一个对象来调用 */
    Test a;
    cout << bind(&Test::sum, &a, 100, 200)() << endl;
    cout << bind(&Test::sum, Test(), 100, 200)() << endl;
    /* bind 还支持参数占位符(最多20个) 可以指定某个参数由用户传递 */
    bind(hello, placeholders::_1)("hello bind placeholders::_1"); /* 第一个参数被占位 必须由用户传递 */
    cout << bind(sum, placeholders::_1, placeholders::_2)(1000, 2000) << endl;
    cout << bind(sum, placeholders::_1, 3000)(1000) << endl;
    /* 我们可以将绑定器获得的函数对象 使用function保留下来 */
    function<void()> func1 = bind(hello, "hello bind");
    function<void(string)> func2 = bind(hello, placeholders::_1);
    func1();
    func2("function + bind");
    return 0;
}
```







## lambda 表达式 (c++11)

c++11标准中提供了函数对象的升级版：lambda表达式。

函数对象的缺点：一般使用在泛型算法参数传递过程中，作为自定义的操作。我们要使用函数对象，就需要在代码中定义函数对象类型，但是它往往只用在某个语句中，用法过于繁琐。


lambda表达式的语法：  
- `[捕获外部变量](形参列表)->返回值{操作代码}`；
- 如果返回值为`void`，可以省略`->void`，`[...](...){...}`；
- 捕获外部变量的方式也有几种：
  - `[]`，不捕获外部变量；
  - `[=]`，以传值方式捕获外部所有变量；
  - `[&]`，以传引用方式捕获外部所有变量；
  - `[this]`，捕获外部this指针；
  - `[=, &a]`，以传值方式捕获外部所有变量，但是`a`变量以传引用方式捕获；
  - `[a, b]`，以传值方式捕获`a`和`b`变量；
  - `[a, &b]`，以传值方式捕获`a`，传引用方式捕获`b`。

注意，lambda表达式用在语句中，如果我们想跨语句使用之前定义的lambda表达式，应该用`function`函数对象类型来保存。


### lambda 的实现原理

lambda表达式实际上就是用一种更简洁的语法，**让编译器为我们提供函数对象类型的定义**。

示例：  
```C++
#include <iostream>
using namespace std;

/* 语句#1 实际上就相当于编译器帮我们定义了TestLambda01类型 */
template<typename T=void>
class TestLambda01 {
public:
    /* []()->void{...} 这里的[]就代表构造函数不需要参数 也就是该类型没有成员变量 */
    TestLambda01() {}
    void operator()() const { cout << "hello lambda" << endl; }
};

/* 语句#2 编译器定义的类型 */
template<typename T=int>
class TestLambda02 {
public:
    TestLambda02() {}
    int operator()(int a, int b) const { return a + b; }
};

/* 语句#3 编译器定义的类型 */
template<typename T=int>
class TestLambda03 {
public:
    /* 捕获了两个引用变量 */
    TestLambda03(int& a, int& b) : _ma(a), _mb(b) {}
    void operator()() const {
        int temp = _ma;
        _ma = _mb;
        _mb = _ma;
    }
private:
    int& _ma;
    int& _mb;
};

int main() {

    /* #1 */
    auto func1 = []()->void{ cout << "hello lambda" << endl; };
    func1();

    /* #2 */
    auto func2 = [](int a, int b)->int{ return a + b; };
    cout << func2(10, 20) << endl;

    /* #3 */
    int a = 100;
    int b = 200;
    auto func3 = [&a, &b]() {
        int temp = a;
        a = b;
        b = temp;
    };
    func3();
    cout << a << " " << b << endl;
    return 0;
}
/*
xushun@xushun-virtual-machine:~/cppadvanced$ g++ lambda1.cpp && ./a.out
hello lambda
30
200 100
*/
```



### lambda 表达式的应用

在使用泛型算法时，灵活使用lambda表达式，可以无需定义函数对象，写出简洁的算法表示。

使用示例：  
```C++
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;

int main() {
    vector<int> vec;
    vec.reserve(20);
    srand(time(nullptr));
    /* 随机生成20个数 */
    for (int i = 0; i < 20; ++ i) {
        vec.push_back(rand() % 100 + 1);
    }
    /* 输出vec */
    for_each(vec.begin(), vec.end(), [](int a){
        cout << a << " ";
    }); cout << endl;
    /* 将vec按从大到小的顺序排列 */
    sort(vec.begin(), vec.end(), [](int a, int b)->bool{
        return a > b;
    });
    /* 输出vec */
    for_each(vec.begin(), vec.end(), [](int a){
        cout << a << " ";
    }); cout << endl;
    /* 找到第一个小于70的数 在它前面插入70 */
    auto it = find_if(vec.begin(), vec.end(), [](int a)->bool{
        return a < 70;
    });
    vec.insert(it, 70);
    /* 输出vec */
    for_each(vec.begin(), vec.end(), [](int a){
        cout << a << " ";
    }); cout << endl;
    return 0;
}
/*
xushun@xushun-virtual-machine:~/cppadvanced$ g++ lambda2.cpp && ./a.out
37 18 72 72 30 3 14 41 16 53 3 66 83 49 97 74 10 2 69 52 
97 83 74 72 72 69 66 53 52 49 41 37 30 18 16 14 10 3 3 2 
97 83 74 72 72 70 69 66 53 52 49 41 37 30 18 16 14 10 3 3 2 
*/
```


示例2：  
```C++
#include <iostream>
#include <memory>
#include <functional>
#include <queue>
#include <algorithm>
using namespace std;

class Data {
public:
    Data(int a, int b) : _ma(a), _mb(b) {}
    int _ma;
    int _mb;
    friend ostream& operator<<(ostream& out, const Data& d);
};
ostream& operator<<(ostream& out, const Data& d) {
    cout << d._ma << " " << d._mb;
    return out;
}

int main() {
    /* lambda 自定义智能指针的删除器 */
    unique_ptr<FILE, function<void(FILE*)>> 
        ptr1(fopen("data.txt", "w"), [](FILE* pf){
            fclose(pf);
        });
    /* 优先队列 lambda 自定义比较函数 */
    priority_queue<Data, vector<Data>, function<bool(Data&, Data&)>>
        maxheap([](Data& d1, Data& d2)->bool{
            return d1._ma + d1._mb < d2._ma + d2._mb; /* 用_ma+_mb来比较 */
        });
    maxheap.push(Data(10, 20));
    maxheap.push(Data(70, 80));
    maxheap.push(Data(50, 60));
    maxheap.push(Data(30, 40));
    while (maxheap.empty() == false) {
        cout << maxheap.top() << endl;
        maxheap.pop();
    }
    return 0;
}
/*
xushun@xushun-virtual-machine:~/cppadvanced$ g++ lambda3.cpp && ./a.out
70 80
50 60
30 40
10 20
*/
```







