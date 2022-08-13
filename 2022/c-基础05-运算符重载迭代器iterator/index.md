# 【C++基础】05 - 运算符重载、迭代器iterator


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
    // 这样写是正确的 但是在这个过程中又分配了一次相同的空间
    // 效率非常低 c++的迭代器iterator就是用来解决这个问题的
    // 下一节来解决这个问题
    delete[] ptmp;
    return stmp;
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















