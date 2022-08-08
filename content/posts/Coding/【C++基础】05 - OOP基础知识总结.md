---
title: 【C++基础】05 - OOP基础知识总结
date: 2022-08-08
tags: [C++, Linux]
categories: [Coding]
---

本篇总结C++OOP的基础知识。

## 类、对象、this指针

### 类和对象简单介绍
**类**，是实体的抽象类型。

使用OOP解决问题首先要在问题中找到实体，从实体（属性、行为）可以得到抽象数据类型ADT（abstract data type），再从ADT得到C++类。  
从实体属性中得到的，就称为**成员变量**；  
从实体行为中得到的，就称为**成员方法**。

将类**实例化**为**对象**后，它才能表示一个实体。

OOP的四大特征：抽象、封装/隐藏、继承、多态。


示例代码：  
```C++
#include <string.h>
#include <iostream>
using namespace std;
const int NAME_LEN = 20;
class CGoods { // C表示class
// 访问限定符 public公有 private私有 protected保护的  体现了封装的特性
public:
    // 给外部提供公有的方法 来访问私有的属性
    // 成员方法编译后存放在代码段上
    void init(const char* name, double price, int amount) { // 如果要使用常量字符串作为参数 必须使用 const
        strcpy(_name, name);
        _price = price;
        _amount = amount;
    }
    void showInfo(); // 驼峰命名
    void setPrice(double price) { _price = price; }
    void setAmount(int amount);
    // 成员方法可以在类定义中直接定义 也可以在类定义外定义
    // 类定义内实现的方法 自动被处理为inline内联函数
private:
    // 属性一般是私有的 外部不能直接访问
    char _name[NAME_LEN]; // 成员变量使用 '_'开头
    double _price;
    int _amount;
};
void CGoods::showInfo() {
    cout << "name : " << _name << endl;
    cout << "price : " << _price << endl;
    cout << "amount : " << _amount << endl;
}
inline void CGoods::setAmount(int amount) { _amount = amount; }
// 在类定义外定义成员函数
// 需要在返回值后加上类的作用域
// 在类外实现的成员方法就不会自动inline内联了 需要在前面加个inline

int main() {
    CGoods good; // 类实例化了一个对象 栈上
    good.init("cola", 2.5, 100);
    good.showInfo();
    
    // 对象的大小只和成员变量有关 和成员函数无关
    // 和struct内存对齐计算方式相同
    return 0;
}
```

### this指针是什么

类可以定义无数的对象，每个对象都有自己的成员变量（对象存储在堆栈上），但是它们共享一套成员方法（类的成员方法编译后，存储在代码段上）。

但是，当调用成员变量时，例如`good.init("cola", 2.5, 100);  good.showInfo();`时，成员方法是如何区分对象的呢？它是如何确定当前处理的成员变量是谁的成员变量呢？

答：在汇编层次，是不存在OOP概念的，这是高级语言的概念，汇编层面只有普通函数调用，所以要知道对哪个对象进行操作，就需要将**对象的地址作为参数传递**。编译时，所有成员方法的参数列表都会加入一个`this`指针，用于接收调用该方法的对象的地址，例如`void init(CGoods* this, const char* name, double price, int amount) {// ...}`，调用该函数时，调用方式为`init(&good, "cola", 2.5, 100);`。在成员方法中访问成员变量时，编译器会自动加上`this->`，例如`_price = price;`，实际为`this->_price = price;`。

### 练习 - OOP实现一个顺序栈

```C++
#include <iostream>
using namespace std;

class SeqStack {
private:
	void resize() { // 扩容  不希望被外部调用 使用private
		int* ptmp = new int[_size * 2];
		for (int i = 0; i < _size; ++ i) {
			ptmp[i] = _pstack[i];
		}
		delete[] _pstack;
		_pstack = ptmp;
		_size *= 2;
	}
public:
	void init(int size = 10) { // 初始化栈
		_pstack = new int[size];
		_top = 0;
		_size = size;
	}
	void release() { // 释放栈
		delete[] _pstack;
		_pstack = nullptr;
	}
	bool full() { // 是否栈满
		return _top == _size;
	}
	bool empty() { // 是否栈空
		return _top == 0;
	}
	void push(int val) { // 入栈
		if (full()) {
			resize();
		}
		_pstack[_top ++] = val;
	}
	void pop() { // 退栈
		-- _top; // 无需判断栈空 外部使用时搭配empty()
	}
	int top() {
		return _pstack[_top - 1];
	}
private:
	int* _pstack;	// 栈地址
	int _top;		// 栈顶指针
	int _size;		// 容量
};


int main() {
	SeqStack s;
	s.init(5);
	for (int i = 0; i < 15; ++ i) {
		s.push(rand() % 100);
	}
	while (!s.empty()) {
		cout << s.top() << " ";
		s.pop();
	}
	s.release();
	return 0;
}
```
练习代码没使用构造、析构函数，下面介绍。

## 构造函数和析构函数
- 函数名和类名一样（析构函数函数名前加`~`），没有返回值；
- 构造函数可以带参数，可以提供多个构造函数（可重载）；
- 析构函数是不带参数的，每个类都只能有一个析构函数；
- 析构函数可以手动调用，调用析构函数后，对象就不存在了（栈上的内存还在，直到退出作用域，所以析构函数调用后再调用成员方法极有可能导致堆内存的非法访问），建议不要手动析构；
- 当你没有提供构造和析构函数时，编译器会为你生成默认的构造和析构函数（都是空函数，什么都不做）。

栈上对象：  
1. 定义时在栈上开辟内存；
2. 自动调用构造函数；
3. 栈上的对象离开作用域时，会依次（先构造的后析构）自动调用析构函数。

全局对象（`.data`）：  
1. 定义时构造；
2. 程序结束时，析构。

堆上对象：  
1. `new`时构造（先分配堆上对象内存，然后调用构造函数）；
2. `delete`时析构（先调用对象的析构函数，然后释放堆上对象内存）。

上面的示例换成构造、析构版本：  
```C++
SeqStack(int size = 10) {
    _pstack = new int[size];  // 初始化栈
    _top = 0;
    _size = size;
}
~SeqStack() {
    delete[] _pstack;  // 释放栈
    _pstack = nullptr;
}
```


## 对象的深拷贝和浅拷贝












## 类和对象代码应用实践
















## 构造函数初始化列表














## 类的各种成员及区别










## 指向类成员的指针






