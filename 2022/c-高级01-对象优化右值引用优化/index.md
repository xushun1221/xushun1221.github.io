# 【C++高级】01 - 对象优化&右值引用优化


## 对象使用中调用了哪些函数？（构造、析构、拷贝、赋值）

### 对象函数调用过程 - 示例1

```C++
#include <iostream>
using namespace std;

class Test {
public:
    Test(int a = 10) : _ma(a) { cout << "Test(int)" << endl; }
    ~Test() { cout << "~Test()" << endl; }
    Test(const Test& t) : _ma(t._ma) { cout << "Test(const Test&)" << endl; }
    Test& operator=(const Test& t) {
        _ma = t._ma;
        cout << "operator=(cosnt Test&)" << endl;
        return *this;
    }
private:
    int _ma;
};

int main() {
    Test t1;            // 默认构造
    Test t2(t1);        // 拷贝构造
    Test t3 = t1;       // 拷贝构造 不是赋值重载
    Test t4 = Test(20); // 普通构造 相当于 Test t4(20);
/* #1
    输出：
    Test(int)
    Test(const Test&)
    Test(const Test&)
    Test(int)

    Test(20); 意为显示生成一个临时对象
    临时对象的生存周期 仅为当前的语句 出语句后就析构
    那么 朴素的想法是 Test t4 = Test(20);
        1. 首先调用构造生成一个临时对象 Test(int)
        2. 然后调用拷贝构造生成对象t4 Test(const Test&)
        3. 最后出语句调用临时对象的析构 ~Test()
    但是事实并非如此 测试结果表明
    Test t4 = Test(20); 该语句仅调用了普通的构造函数 Test(int)

    这是C++ 编译器提供的优化
    当我们使用临时对象生成一个新对象时 C++编译器会优化这一过程
    不会构造临时对象并调用拷贝构造 而是直接构造新对象
*/
    cout << "-----------------" << endl;

    t4 = t2;        // 赋值重载
    t4 = Test(30);  // 赋值重载  
/* #2
    输出：
    operator=(cosnt Test&)
    Test(int)
    operator=(cosnt Test&)
    ~Test()

    该语句一定会产生临时对象 
    因为t4已经存在了 而非需要构造
    需要先后调用 1.临时对象构造 2.赋值重载 3.临时对象析构
*/
    cout << "-----------------" << endl;

    t4 = (Test)30; // 赋值重载  显式Test <- int 构造函数Test(int) 
    t4 = 30;       // 赋值重载  隐式Test <- int 构造函数Test(int) 
/* #3
    输出： 
    Test(int)
    operator=(cosnt Test&)
    ~Test()
    Test(int)
    operator=(cosnt Test&)
    ~Test()

    当其他类型想要转换为某个类类型时
    编译器会检查该类是否有合适的构造函数
    例如 这里Test类有带有int参数的构造函数 Test(int)
    所以 t4 = (Test)30; t4 = 30; 中的显式和隐式类型转换都可以成功
    由于 t4 对象已经存在 所以需要构造临时对象
        1.构造函数 生成临时对象 2.调用赋值重载 3.在离开语句时 临时对象析构
*/
    cout << "-----------------" << endl;

    // Test* p = &Test(40);     // 错误写法 对象指针不能指向一个临时变量
    const Test& ref = Test(50); // 正确写法 const对象引用 可以指向一个临时变量 (必须使用const引用才能指向临时对象 普通引用不行）
/* #4
    输出：
    Test(int)

    从之前的分析可以知道 临时变量的生存周期就是这条语句 出语句后就会析构
    所以 我们不能用一个指针指向临时变量
    但是 可以用一个const引用指向临时变量
    这是为什么呢？
        临时变量没有名字 出了语句后无法再使用它 所以出语句后就析构
        但是 引用相当于给这块内存（对象）又赋了一个名字
        那么出了语句后 仍然可以使用该对象
        这个临时对象的生命周期 就变成了引用对象的生命周期
*/
    cout << "-----------------" << endl;
/*
    输出：
    ~Test()
    ~Test()
    ~Test()
    ~Test()
    ~Test()
*/
    return 0;
}
```

### 对象函数调用过程 - 示例2

```C++
#include <iostream>
using namespace std;

class Test {
public:
    Test(int a = 5, int b = 5) : _ma(a), _mb(b) { cout << "Test(int, int)" << endl; }
    /* Test()    Test(int)    Test(int, int)    这三种都可以*/
    ~Test() { cout << "~Test()" << endl; }
    Test(const Test& t) : _ma(t._ma), _mb(t._mb) { cout << "Test(const Test&)" << endl; }
    Test& operator=(const Test& t) {
        _ma = t._ma;
        _mb = t._mb;
        cout << "operator=(const Test&)" << endl;
        return *this;
    }
private:
    int _ma;
    int _mb;
};

Test t1 (10, 10); // #1 普通构造 Test(int, int)
/* 
    程序运行时 全局变量先进行构造
    全局变量在main函数之前就要进行构造
*/
int main() {

    Test t2(20, 20);     // #3 普通构造 Test(int, int)
    Test t3 = t2;        // #4 拷贝构造 Test(const Test&)
    static Test t4 = Test(30, 30);  // #5 普通构造 相当于 static Test t4(30, 30);
    /*
        上面说过 临时对象拷贝构造新对象会被优化为直接构造新对象
        static 修饰的变量 在程序开始时就在数据段分配了内存
        直到运行到该语句时 才会进行对象构造
        数据段上的对象 在程序运行结束后才析构
    */
    t2 = Test(40, 40);      // #6 赋值重载 生成了临时对象 Test(int, int) operator=(const Test&) ~Test()
    t2 = (Test)(50, 50);    // #7 赋值重载 生成了临时对象 Test(int) operator=(const Test&) ~Test()
    /* 
        t2 = (Test)(50, 50); 等号右边有两个部分
        (50, 50) 是一个逗号表达式 逗号表达式的值是表达式中最后一个表达式的值 这里就是50
        (Test) 是强制类型转换
        所以该语句相当于 t2 = (Test)50; 将50转换为 Test 类型的对象
        编译器可以找到这样的构造函数 Test(int)
    */
    t2 = 60; // #8 赋值重载 生成了临时对象 Test(int) operator=(const Test&) ~Test()
    /* 和 #7 一样 */
    Test* p1 = new Test(70, 70);    // #9 Test(int, int)
    /*
        new 构造的是堆上对象 不是临时对象
        堆上对象 只有在delete时 才会析构
    */
    Test* p2 = new Test[2];         // #10 堆上对象数组 Test() Test()
    /* 数组中每个对象都要构造 */
    // Test* p3 = &Test(80, 90);    // #11 错误写法 
    /* 指针不能指向临时对象 */
    const Test& p4 = Test(90, 90);  // #12 Test(int, int)
    /* const引用指向的临时对象 语句结束后不会立即析构 */
    delete p1;   // #13 ~Test()
    delete[] p2; // #14 ~Test() ~Test()

    return 0;
}
Test t5(100, 100); // #2 普通构造 Test(int, int)
/* 虽然写在main后面 但仍然先于main进行构造 */
```


## 函数调用过程中对象背后调用的方法太多！

示例：  
```C++
#include <iostream>
using namespace std;

class Test {
public:
    Test(int a = 10) : _ma(a) { cout << "Test(int)" << endl; }
    ~Test() { cout << "~Test()" << endl; }
    Test(const Test& t) : _ma(t._ma) { cout << "Test(const Test&)" << endl;}
    Test& operator=(const Test& t) { 
        _ma = t._ma; 
        cout << "operator=(const Test&)" << endl;
        return *this;
    }
    int getData() const { return _ma; }
private:
    int _ma;
};

Test getObject(Test t) {   // #3 拷贝构造 Test(const Test&)
    int val = t.getData();
    Test tmp(val);  // #4 Test(int)
    return tmp;  // 函数返回的对象也需要仔细分析
    /*
        tmp 是在 getObject 函数栈帧上的一个对象 （局部的对象）
        getObject 函数执行完成后 tmp 对象就被析构了
        它不能被直接带回到 main 函数中

        这里的做法是：
            #5 Test(const Test&)
            在 main 函数栈帧中构造一个临时对象 （通过对 tmp 对象的拷贝构造）

        返回的对象构造完成后 还要把 getObject 函数栈帧中的对象进行析构
        #6 ~Test() tmp 析构
        #7 ~Test() 形参 t 析构

        至此 getObject 函数执行完成
    */
}

int main() {

    Test t1;  // #1 Test(int)
    Test t2;  // #2 Test(int)
    t2 = getObject(t1); // 函数调用过程需要仔细分析
    /*
        1. 对象实参传递给形参的过程是 *初始化* 的过程
            t1 是一个已经存在的对象而 形参是正在构造的对象 
            所以这是一个构造过程 需要调用拷贝构造函数进行构造
        2. getObject 函数执行完成后
            main 函数栈帧中就构造了函数返回的临时对象
            然后使用该临时对象 给 t2 赋值
            #8 operator=(const Test&)
        3. 赋值完成后 该语句结束 临时对象也要被析构
            #9 ~Test()
    */
    return 0;
    // #10 ~Test() t2 析构
    // #11 ~Test() t1 析构
}

/*
我的g++编译器输出的结果是这样的
和上面的分析有一个地方不一样
可能编译器进行了优化

xushun@xushun-virtual-machine:~/cppadvanced$ g++ objtest3.cpp && ./a.out
Test(int)
Test(int)
Test(const Test&)
Test(int)
operator=(const Test&)   [这里没有在main函数栈帧上构造临时对象 而是直接将tmp对象赋值给t2]
~Test()
~Test()
~Test()
~Test()
*/
```




## 三条对象优化的规则

1. 函数参数传递过程中，对象应该**按引用传递**，不要按值传递；
2. 函数返回对象的时候，应该**返回一个临时对象**，不要返回一个定义过的对象；
3. 接收返回值是对象的函数调用的时候，应该**按初始化的方式接收**，不要按赋值的方式接收。

注意，下面这句话非常重要！

>用临时对象，拷贝构造一个新对象时，编译器会跳过构造临时对象的过程，直接构造新对象。




示例：  
```C++
#include <iostream>
using namespace std;

class Test {
public:
    Test(int a = 10) : _ma(a) { cout << "Test(int)" << endl; }
    ~Test() { cout << "~Test()" << endl; }
    Test(const Test& t) : _ma(t._ma) { cout << "Test(const Test&)" << endl;}
    Test& operator=(const Test& t) { 
        _ma = t._ma; 
        cout << "operator=(const Test&)" << endl;
        return *this;
    }
    int getData() const { return _ma; }
private:
    int _ma;
};

// Test getObject(Test t) {
Test getObject(Test& t) {   // 函数传参 对象应*按引用传递*
    int val = t.getData();
    /* Test tmp(val);
    return tmp; */
    return Test(val);
    /*
        原来返回一个定义过的对象 需要在 main 函数栈帧中构造一个临时对象
        并由要返回的对象进行拷贝构造

        现在返回的是一个临时对象
        我们要 *用临时对象拷贝构造一个新对象（在main栈帧中的临时对象）*
        编译器会跳过构造临时对象的过程 而直接构造新对象

        这里没有生成临时对象 也自然不需要进行析构
        形参 t 按引用传递 也不需要进行析构
    */
}

int main() {

    Test t1;    // #1 Test(int)
    /* Test t2;
    t2 = getObject(t1); */
    Test t2 = getObject(t1);  // 应该按初始化的方式接收函数返回的对象
    /*
        1. t1 按引用传递 不需要进行拷贝构造
        2. 原来用给一个已经定义的对象赋值的方式接收 函数返回的对象时
            需要在 main 的栈帧中 构造一个临时对象 再进行赋值
            现在 我们使用 初始化方式 接收 返回的对象
            由于*用 main 栈帧中新构造的临时对象 构造一个新对象（t2）*
            编译器会跳过 构造临时对象的过程（不会在 main 栈帧中构造临时对象）
            而是从 return Test(val); 语句处 直接构造 t2 对象
            （实际上 在函数实参压栈时 t2 的地址也作为参数进行压栈了 这样 getObject 函数才能直接构造 t2 对象）
    */
    return 0;
    // #3 ~Test()  t2
    // #4 ~Test()  t1
}

/* 输出
从程序输出可以看出 程序调用的函数大幅减少了 性能得到了很好的优化

xushun@xushun-virtual-machine:~/cppadvanced$ g++ objtest4.cpp && ./a.out
Test(int)
Test(int)
~Test()
~Test()
*/
```





## MyString 存在的问题

分析`MyString`类的实现，分析内存使用效率问题。

```C++
#include <iostream>
#include <cstring>
using namespace std;

class MyString {
public:
    MyString(const char* str = nullptr) {
        cout << "MyString(const char*)" << endl;
        if (str != nullptr) {
            _mptr = new char[strlen(str) + 1];
            strcpy(_mptr, str);
        } else {
            _mptr = new char[1];
            _mptr[0] = '\0';
        }
    }
    ~MyString() {
        cout << "~MyString()" << endl;
        delete[] _mptr;
        _mptr = nullptr;
    }
    MyString(const MyString& str) {
        cout << "MyString(const MyString&)" << endl;
        _mptr = new char[strlen(str._mptr) + 1];
        strcpy(_mptr, str._mptr);
    }
    MyString& operator=(const MyString& str) {
        cout << "operator=(const MyString&)" << endl;
        if (&str == this) {
            return *this;
        }
        delete[] _mptr;
        _mptr = new char[strlen(str._mptr) + 1];
        strcpy(_mptr, str._mptr);
        return *this;
    }
    const char* c_str() const { return _mptr; }
private:
    char* _mptr;
};


// 没有用到上一节说的对象优化方法是为了模拟这种避免不了的情况
MyString GetString(MyString& str) {
    const char* pstr = str.c_str();
    MyString tmpStr(pstr); // #3 普通构造 MyString(const char*)
    return tmpStr;
    /*
        #4 MyString(const MyString&)
            这里要返回一个对象 tmpStr
            在 main 函数栈帧中拷贝构造一个临时对象
        #5 ~MyString()
            GetString 结束 tmpStr 析构

        **这里就有一个很大的问题**
            1. main 中的临时对象 进行了拷贝构造
                重新在堆上分配了一块内存 将 tmpStr _mptr 指向的字符串 拷贝过来
            2. tmpStr _mptr 指向的字符串 拷贝给了临时对象后 立即被释放了
            这样的内存使用效率太低
            新内存块分配、内存拷贝、旧内存块释放
        能不能 直接将 tmpStr的_mptr 转移给 临时对象 而不再进行多次内存操作？
            是可以的 需要通过右值引用来实现
    */
}

int main() {

    MyString str1("aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa");
    // #1 普通构造 MyString(const char*)
    MyString str2; // #2 普通构造 MyString(const char*)
    str2 = GetString(str1);
    /*
        参数按引用传递 无需拷贝构造
        #6 operator=(const MyString&)
            str2 被返回的临时对象赋值
        #7 ~MyString()  临时对象析构

        **这里也存在相同的问题**
            对 str2 进行赋值 需要释放 _mptr 指向的内存 
            再复制临时对象 _mptr 指向的内存
            再析构临时对象 释放 临时对象 _mptr 指向的内存
            效率非常低
    */
    cout << str2.c_str() << endl;
    return 0;
    // #8 ~MyString()  str2 析构
    // #9 ~MyString()  str1 析构
}
```




## 带右值引用参数的拷贝构造和赋值函数

我们可以通过给类的成员方法提供对应的**带右值引用参数的版本**，提高对临时对象操作时的效率问题。


```C++
#include <iostream>
#include <cstring>
using namespace std;

class MyString {
public:
    MyString(const char* str = nullptr) {
        cout << "MyString(const char*)" << endl;
        if (str != nullptr) {
            _mptr = new char[strlen(str) + 1];
            strcpy(_mptr, str);
        } else {
            _mptr = new char[1];
            _mptr[0] = '\0';
        }
    }
    ~MyString() {
        cout << "~MyString()" << endl;
        delete[] _mptr;
        _mptr = nullptr;
    }
    /* 左值引用参数的拷贝构造 */
    MyString(const MyString& str) {
        cout << "MyString(const MyString&)" << endl;
        _mptr = new char[strlen(str._mptr) + 1];
        strcpy(_mptr, str._mptr);
    }
    /* 右值引用参数的拷贝构造 */
    MyString(MyString&& str) { /* 使用临时对象进行拷贝构造时  就会调用该函数 */
        cout << "MyString(MyString&&)" << endl;
        _mptr = str._mptr;
        str._mptr = nullptr;
        /*
            因为临时对象拷贝构造完新对象后就要被析构 它持有的资源无需释放
            直接交给由它构造的新对象即可
            这样可以避免内存开辟和拷贝的开销
        */
    }
    /* 左值引用参数的赋值重载函数 */
    MyString& operator=(const MyString& str) {
        cout << "operator=(const MyString&)" << endl;
        if (&str == this) {
            return *this;
        }
        delete[] _mptr;
        _mptr = new char[strlen(str._mptr) + 1];
        strcpy(_mptr, str._mptr);
        return *this;
    }
    /* 右值引用参数的赋值重载函数 */
    MyString& operator=(MyString&& str) { /* 使用临时对象进行赋值时 就会调用该函数 */
        cout << "operator=(MyString&&)" << endl;
        if (&str == this) {
            return *this;
        }
        delete[] _mptr;
        _mptr = str._mptr;
        str._mptr = nullptr;
        /* 临时对象直接将资源交给该对象即可 */
        return *this;
    }
    const char* c_str() const { return _mptr; }
private:
    char* _mptr;
};


MyString GetString(MyString& str) {
    const char* pstr = str.c_str();
    MyString tmpStr(pstr);
    return tmpStr;
}

int main() {

    MyString str1("aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa");
    MyString str2;
    str2 = GetString(str1);
    cout << str2.c_str() << endl;
    return 0;
}
```


### MyString operator+() 的问题

```C++
#include <iostream>
#include <cstring>
using namespace std;

class MyString {
public:
    MyString(const char* str = nullptr) {
        cout << "MyString(const char*)" << endl;
        if (str != nullptr) {
            _mptr = new char[strlen(str) + 1];
            strcpy(_mptr, str);
        } else {
            _mptr = new char[1];
            _mptr[0] = '\0';
        }
    }
    ~MyString() {
        cout << "~MyString()" << endl;
        delete[] _mptr;
        _mptr = nullptr;
    }
    MyString(const MyString& str) {
        cout << "MyString(const MyString&)" << endl;
        _mptr = new char[strlen(str._mptr) + 1];
        strcpy(_mptr, str._mptr);
    }
    /* 右值引用参数的拷贝构造 */
    MyString(MyString&& str) {
        cout << "MyString(MyString&&)" << endl;
        _mptr = str._mptr;
        str._mptr = nullptr;
    }
    MyString& operator=(const MyString& str) {
        cout << "operator=(const MyString&)" << endl;
        if (&str == this) {
            return *this;
        }
        delete[] _mptr;
        _mptr = new char[strlen(str._mptr) + 1];
        strcpy(_mptr, str._mptr);
        return *this;
    }
    /* 右值引用参数的赋值重载函数 */
    MyString& operator=(MyString&& str) {
        cout << "operator=(MyString&&)" << endl;
        if (&str == this) {
            return *this;
        }
        delete[] _mptr;
        _mptr = str._mptr;
        str._mptr = nullptr;
        return *this;
    }
    const char* c_str() const { return _mptr; }
    friend MyString operator+(const MyString& lhs, const MyString& rhs);
private:
    char* _mptr;
};

MyString operator+(const MyString& lhs, const MyString& rhs) {
    /* v1
        char* ptmp = new char[strlen(lhs._mptr) + strlen(rhs._mptr) + 1];
        strcpy(ptmp, lhs._mptr);
        strcat(ptmp, rhs._mptr);
        return MyString(ptmp);
            // ptmp 内存泄漏了！
    */
    /* v2
        char* ptmp = new char[strlen(lhs._mptr) + strlen(rhs._mptr) + 1];
        strcpy(ptmp, lhs._mptr);
        strcat(ptmp, rhs._mptr);
        MyString tmpStr(ptmp);
        delete[] ptmp; // 内存没泄露 但是效率很低
        return tmpStr;
    */
    /* v3 */
    MyString tmpStr;
    tmpStr._mptr = new char[strlen(lhs._mptr) + strlen(rhs._mptr) + 1];
    strcpy(tmpStr._mptr, lhs._mptr);
    strcat(tmpStr._mptr, rhs._mptr);
    return tmpStr;
    /*
        这里返回的临时对象用于构造 str3 时
        会调用 带右值引用参数的 拷贝构造函数
        内存效率更高
    */
}

int main() {

    MyString str1("hello");
    MyString str2(" world");
    MyString str3 = str1 + str2;
    cout << str3.c_str() << endl;
    return 0;
}
```


### MyString 在 vector 上的应用

```C++
#include <iostream>
#include <cstring>
#include <vector>
using namespace std;

class MyString {
public:
    MyString(const char* str = nullptr) {
        cout << "MyString(const char*)" << endl;
        if (str != nullptr) {
            _mptr = new char[strlen(str) + 1];
            strcpy(_mptr, str);
        } else {
            _mptr = new char[1];
            _mptr[0] = '\0';
        }
    }
    ~MyString() {
        cout << "~MyString()" << endl;
        delete[] _mptr;
        _mptr = nullptr;
    }
    MyString(const MyString& str) {
        cout << "MyString(const MyString&)" << endl;
        _mptr = new char[strlen(str._mptr) + 1];
        strcpy(_mptr, str._mptr);
    }
    MyString(MyString&& str) {
        cout << "MyString(MyString&&)" << endl;
        _mptr = str._mptr;
        str._mptr = nullptr;
    }
    MyString& operator=(const MyString& str) {
        cout << "operator=(const MyString&)" << endl;
        if (&str == this) {
            return *this;
        }
        delete[] _mptr;
        _mptr = new char[strlen(str._mptr) + 1];
        strcpy(_mptr, str._mptr);
        return *this;
    }
    MyString& operator=(MyString&& str) {
        cout << "operator=(MyString&&)" << endl;
        if (&str == this) {
            return *this;
        }
        delete[] _mptr;
        _mptr = str._mptr;
        str._mptr = nullptr;
        return *this;
    }
    const char* c_str() const { return _mptr; }
    friend MyString operator+(const MyString& lhs, const MyString& rhs);
private:
    char* _mptr;
};

MyString operator+(const MyString& lhs, const MyString& rhs) {
    MyString tmpStr;
    tmpStr._mptr = new char[strlen(lhs._mptr) + strlen(rhs._mptr) + 1];
    strcpy(tmpStr._mptr, lhs._mptr);
    strcat(tmpStr._mptr, rhs._mptr);
    return tmpStr;
}

int main() {
    MyString str1 = "aaa"; // 普通构造
    vector<MyString> vec;
    vec.reserve(10);
    cout << "------------------------" << endl;
    vec.push_back(str1); // 拷贝构造
    vec.push_back(MyString("bbb")); // 临时对象普通构造  
    /* 
        push_back 使用临时对象 构造新对象时 调用了 带右值引用参数的拷贝构造 
        直接转移指向堆内存的所有权

        push_back 是怎么做到这一点的？
        下面学习 move移动语义 和 forward完美转发
    */
    // 临时对象析构
    cout << "------------------------" << endl;
    return 0;
}
/*
xushun@xushun-virtual-machine:~/cppadvanced$ g++ mystring4.cpp && ./a.out
MyString(const char*)
------------------------
MyString(const MyString&)
MyString(const char*)
MyString(MyString&&)
~MyString()
------------------------
~MyString()
~MyString()
~MyString()
*/
```




## move 移动语义 forward 完美转发


### 实现右值引用的 vector::push_back()

这里我们使用`std::move()`，实现了和标准库一样的，带右值引用参数的`vector::push_back()`函数。

`move`可以进行**移动语义**。

```C++
#include <iostream>
#include <cstring>
using std::cout;
using std::endl;

template<typename T>
class allocator {
public:
    T* allocate(size_t size) {
        return (T*)malloc(sizeof(T) * size);
    }
    void deallocate(void* p) { 
        free(p);
    }
    /* 左值引用参数 */
    void construct(T* p, const T& val) { 
        new (p) T(val);  /* 定位 new */
    }
    /* 为 construct 踢狗带右值引用参数的版本 */
    void construct(T* p, T&& val) {
        /*
            这里的 val 还是左值（右值引用本身是左值）
            new (p) T(val);
            上面这句会调用 带左值引用的拷贝构造函数
        */
        new (p) T(std::move(val));
        /* 需要进行类型转换 转为右值引用 */
    }
    void destroy(T* p) { 
        p->~T(); 
    }
};

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
    /* 左值引用参数 */
    void push_back(const T& val) {
        if (full()) {
            expand();
        }
        _allocator.construct(_last, val);
        ++ _last;
    }
    /* 为 push_back 添加带右值引用参数的版本 */
    void push_back(T&& val) {
        if (full()) {
            expand();
        }
        /* 
            一个右值引用变量的本身还是一个左值 
            _allocator.construct(_last, val);
            所以 上面这一句匹配的仍然是 void construct(T* p, const T& val);
        */
        _allocator.construct(_last, std::move(val));
        /*
            我们需要使用 std::move(val) 将左值引用类型转换为 右值引用
            这样就可以调用 带右值引用参数的 construct 函数
        */
        ++ _last;
    }
    void pop_back() {
        if (!empty()) {
            -- _last;
            _allocator.destroy(_last);
        }
    }
    T back() const {
        return *(_last - 1);
    }
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
private:
    T* _first;
    T* _last;
    T* _end;
    Alloc _allocator; 
};

class MyString {
public:
    MyString(const char* str = nullptr) {
        cout << "MyString(const char*)" << endl;
        if (str != nullptr) {
            _mptr = new char[strlen(str) + 1];
            strcpy(_mptr, str);
        } else {
            _mptr = new char[1];
            _mptr[0] = '\0';
        }
    }
    ~MyString() {
        cout << "~MyString()" << endl;
        delete[] _mptr;
        _mptr = nullptr;
    }
    MyString(const MyString& str) {
        cout << "MyString(const MyString&)" << endl;
        _mptr = new char[strlen(str._mptr) + 1];
        strcpy(_mptr, str._mptr);
    }
    MyString(MyString&& str) {
        cout << "MyString(MyString&&)" << endl;
        _mptr = str._mptr;
        str._mptr = nullptr;
    }
    MyString& operator=(const MyString& str) {
        cout << "operator=(const MyString&)" << endl;
        if (&str == this) {
            return *this;
        }
        delete[] _mptr;
        _mptr = new char[strlen(str._mptr) + 1];
        strcpy(_mptr, str._mptr);
        return *this;
    }
    MyString& operator=(MyString&& str) {
        cout << "operator=(MyString&&)" << endl;
        if (&str == this) {
            return *this;
        }
        delete[] _mptr;
        _mptr = str._mptr;
        str._mptr = nullptr;
        return *this;
    }
    const char* c_str() const { return _mptr; }
    friend MyString operator+(const MyString& lhs, const MyString& rhs);
private:
    char* _mptr;
};
MyString operator+(const MyString& lhs, const MyString& rhs) {
    MyString tmpStr;
    tmpStr._mptr = new char[strlen(lhs._mptr) + strlen(rhs._mptr) + 1];
    strcpy(tmpStr._mptr, lhs._mptr);
    strcat(tmpStr._mptr, rhs._mptr);
    return tmpStr;
}

int main() {
    MyString str1 = "aaa";
    vector<MyString> vec;
    cout << "------------------------" << endl;
    vec.push_back(str1);
    vec.push_back(MyString("bbb")); // 临时对象普通构造  
    cout << "------------------------" << endl;
    return 0;
}
/*
xushun@xushun-virtual-machine:~/cppadvanced$ g++ mystring5.cpp && ./a.out
MyString(const char*)
------------------------
MyString(const MyString&)
MyString(const char*)
MyString(MyString&&)
~MyString()
------------------------
~MyString()
~MyString()
~MyString()
*/
```


### forward 类型完美转发


```C++
#include <iostream>
#include <cstring>
using std::cout;
using std::endl;

template<typename T>
class allocator {
public:
    T* allocate(size_t size) {
        return (T*)malloc(sizeof(T) * size);
    }
    void deallocate(void* p) { 
        free(p);
    }
    /*
        void construct(T* p, const T& val) { 
            new (p) T(val);
        }
        void construct(T* p, T&& val) {
            new (p) T(std::move(val));
        }
    */
    /* 将 construct 的左值参数和右值参数版本合并编写 */
    template<typename Ty>
    void construct(T* p, Ty&& val) {
        new (p) T(std::forward<Ty>(val)); /* 类型转发 */
    }
    void destroy(T* p) { 
        p->~T(); 
    }
};

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
    // void push_back(const T& val) {
    //     if (full()) {
    //         expand();
    //     }
    //     _allocator.construct(_last, val);
    //     ++ _last;
    // }
    // void push_back(T&& val) {
    //     if (full()) {
    //         expand();
    //     }
    //     _allocator.construct(_last, std::move(val));
    //     ++ _last;
    // }
    /* 将 push_back 的左值参数和右值参数版本合并编写 */
    template<typename Ty>
    void push_back(Ty&& val) {
        /*
            Ty&& val  既可以接收左值 也可以接收右值
            这是 c++ 的 引用折叠 提供的特性
            1. 如果实参是左值 应该用左值引用来引用它 即 Ty&
                Ty& + && = Ty&     那么 val 还是一个左值引用
            2. 如果实参是右值 应该用右值引用来引用它 即 Ty&&
                Ty&& + && = Ty&&   那么 val 还是一个右值引用

            函数模板的类型推演 + 引用折叠
            可以合并两个版本的 push_back 函数
        */
        if (full()) {
            expand();
        }
        /*
            _allocator.construct(_last, val);
            无论 val 是左值引用 还是右值引用
            val 本身都还是一个左值
            无法调用到 右值引用参数的 construct 函数
        */
        _allocator.construct(_last, std::forward<Ty>(val));
        /*
            使用 std::forward<Ty>(val) 可以自动进行类型完美转发
            1. 如果 val 是左值引用 std::forward<Ty>(val) 就是左值
            2. 如果 val 是右值引用 std::forward<Ty>(val) 就是右值
        */
        ++ _last;
    }
    void pop_back() {
        if (!empty()) {
            -- _last;
            _allocator.destroy(_last);
        }
    }
    T back() const {
        return *(_last - 1);
    }
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
private:
    T* _first;
    T* _last;
    T* _end;
    Alloc _allocator; 
};



class MyString {
public:
    MyString(const char* str = nullptr) {
        cout << "MyString(const char*)" << endl;
        if (str != nullptr) {
            _mptr = new char[strlen(str) + 1];
            strcpy(_mptr, str);
        } else {
            _mptr = new char[1];
            _mptr[0] = '\0';
        }
    }
    ~MyString() {
        cout << "~MyString()" << endl;
        delete[] _mptr;
        _mptr = nullptr;
    }
    MyString(const MyString& str) {
        cout << "MyString(const MyString&)" << endl;
        _mptr = new char[strlen(str._mptr) + 1];
        strcpy(_mptr, str._mptr);
    }
    MyString(MyString&& str) {
        cout << "MyString(MyString&&)" << endl;
        _mptr = str._mptr;
        str._mptr = nullptr;
    }
    MyString& operator=(const MyString& str) {
        cout << "operator=(const MyString&)" << endl;
        if (&str == this) {
            return *this;
        }
        delete[] _mptr;
        _mptr = new char[strlen(str._mptr) + 1];
        strcpy(_mptr, str._mptr);
        return *this;
    }
    MyString& operator=(MyString&& str) {
        cout << "operator=(MyString&&)" << endl;
        if (&str == this) {
            return *this;
        }
        delete[] _mptr;
        _mptr = str._mptr;
        str._mptr = nullptr;
        return *this;
    }
    const char* c_str() const { return _mptr; }
    friend MyString operator+(const MyString& lhs, const MyString& rhs);
private:
    char* _mptr;
};
MyString operator+(const MyString& lhs, const MyString& rhs) {
    MyString tmpStr;
    tmpStr._mptr = new char[strlen(lhs._mptr) + strlen(rhs._mptr) + 1];
    strcpy(tmpStr._mptr, lhs._mptr);
    strcat(tmpStr._mptr, rhs._mptr);
    return tmpStr;
}


int main() {
    MyString str1 = "aaa";
    vector<MyString> vec;
    cout << "------------------------" << endl;
    vec.push_back(str1);
    vec.push_back(MyString("bbb"));
    cout << "------------------------" << endl;
    return 0;
}
/*
xushun@xushun-virtual-machine:~/cppadvanced$ g++ mystring6.cpp && ./a.out
MyString(const char*)
------------------------
MyString(const MyString&)
MyString(const char*)
MyString(MyString&&)
~MyString()
------------------------
~MyString()
~MyString()
~MyString()
*/
```
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       







