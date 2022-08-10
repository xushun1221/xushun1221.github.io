# 【C++基础】03 - OOP基础知识总结


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
		for (int i = 0; i < _size; ++ i) { // 这里为什么不用memcpy、realloc等内存拷贝函数呢？
			ptmp[i] = _pstack[i];		   // 在oop编程中要谨慎使用内存拷贝 因为有可能导致浅拷贝的问题
		}								   // 这里我们拷贝的内存只保护int内容 所以是可以使用内存拷贝函数的
		delete[] _pstack;				   // 但是如果拷贝的内存中还包括一些指向外部资源（如堆内存）的指针
		_pstack = ptmp;					   // 那就必须进行深拷贝
		_size *= 2;						   // 否则在delete原来的内存时 有可能释放掉外部资源 导致新内存中指向外部资源的指针变成野指针
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
- 当你没有提供构造和析构函数时，编译器会为你生成默认的构造和析构函数（都是空函数，什么都不做），当你提供了构造函数后，编译器不会再提供默认构造函数。

栈上对象：  
1. 定义时在栈上开辟内存；
2. 自动调用构造函数；
3. 栈上的对象离开作用域时，会依次（先构造的后析构）自动调用析构函数。

全局对象（`.data`）：  
1. 程序开始时，构造；
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


## 对象的浅拷贝和深拷贝

我们来分析一下下面的代码：  
```C++
class SeqStack {
public:
	SeqStack(int size = 10) {
		_pstack = new int[size];  // 初始化栈
		_top = 0;
		_size = size;
	}
	~SeqStack() {
		delete[] _pstack;  // 释放栈
		_pstack = nullptr;
	}
private:
	int* _pstack;	// 栈地址
	int _top;		// 栈顶指针
	int _size;		// 容量
};
int main() {
	SeqStack s1(10);  // #1
	SeqStack s2 = s1; // #2 // 析构s2时会崩溃
	// SeqStack s3(s1);  // #3 同 #2
	s2 = s1; // #4 赋值操作 s2.operator=(s1);
	return 0;
}
```
1. `SeqStack s1(10);`，在栈上开辟了一个`SeqStack`对象大小的内存空间，然后调用构造函数进行初始化，构造函数又在堆上开辟了一个10`int`大小的内存空间，`_pstack`指针指向这片堆上内存；
2. `SeqStack s2 = s1;`，用同类型的对象产生一个新的对象，会调用**拷贝构造函数**（如果没有提供拷贝构造函数，编译器会提供一个默认的拷贝构造函数）；
3. `SeqStack s3(s1);`，同样也是使用默认的拷贝构造函数，同上；
4. `s2 = s1;`，对象的赋值操作，如果我们没有自定义赋值操作（重载赋值运算符），编译器就会提供**默认赋值函数**。

### 浅拷贝
默认的拷贝构造函数，对内存进行拷贝，例如`SeqStack s2 = s1;`，在栈上分配一个`SeqStack`对象的空间，然后将`s1`对象的内存直接拷贝到新分配的内存上，完成`s2`的构造。这样存在一个严重的问题，`s2`中的`_pstack`和`s1`中的相同，它们都指向堆上的同一块内存，对新对象`s2`的操作会直接影响`s1`，而且在离开作用域进行析构时，`s2  s1`的析构会导致`_pstack`指向的内存被`delete`两次，造成程序崩溃。  
这个过程就是**浅拷贝**（对象直接进行内存拷贝）。

对象的浅拷贝不是一定会出错，出错的原因是对象的某些成员变量（尤其是指针），指向了对象内存之外的外部资源（如堆内存等），当浅拷贝发生，多个对象指向了同一个外部资源，导致相关的错误。

这种情况下，我们就不能依赖编译器提供的默认拷贝构造函数。

注意，默认的赋值函数，同样进行内存拷贝，也是浅拷贝，所以赋值函数也许需要自定义。

### 深拷贝
自定义的拷贝构造，实现深拷贝：  
```C++
SeqStack(const SeqStack& src) {
	_pstack = new int[src._size];
	for (int i = 0; i < src.top; ++ i) {
		_pstack[i] = src._pstack[i];
	}
	_top = src._top;
	_size = src._size;
}
```

自定义的赋值函数（重载赋值运算符），实现深拷贝：  
```C++
SeqStack& operator=(const SeqStack& src) {
	// 自己给自己赋值 是存在风险且没必要的操作
	if (this == &src) {
		return *this;
	}
	// 已存在的对象才能进行赋值 所以原来的资源需要释放掉
	delete[] _pstack;
	_pstack = new int[src._size];
	for (int i = 0; i < src.top; ++ i) {
		_pstack[i] = src._pstack[i];
	}
	_top = src._top;
	_size = src._size;
	return *this; // 赋值运算符需要能连续赋值 s3 = s2 = s1; s3.operator=(s2.operator=(s1));
}
```



## 构造函数的初始化列表

### 初始化列表
示例代码：  
```C++
#include <iostream>
#include <cstring>
using namespace std;
// 日期类
class CDate {
public:
	CDate(int year, int month, int day) {
		_year = year;
		_month = month;
		_day = day;
	}
	void show() {
		cout << _year << "-" << _month << "-" << _day << endl;
	}
private:
	int _year;
	int _month;
	int _day;
};

// 货物类
class CGoods {
public:
	CGoods(const char* name, int price, int amount, int year, int month, int day) 
			: _date(year, month, day), _price(price), _amount(amount) { // 初始化列表
		strcpy(_name, name); // 一般函数不能放在初始化列表中
		// _date = CDate(year, month, day);
		// 这是错误写法 原因是
		// 在初始化列表中的构造函数直接指定了三个参数的构造函数
		// 如果 _date = CDate(year, month, day);
		// 说明 _date 已存在 编译器要使用默认构造 
		// 但是 我们已经提供了构造函数 编译器就无法使用默认构造 所以会出错
	}
	void show() {
		cout << "name : " << _name << endl;
		cout << "price : " << _price << endl;
		cout << "amount : " << _amount << endl;
		cout << "date : "; _date.show();
	}
private:
	char _name[20];
	int _price;
	int _amount;
	CDate _date; // CDate类的一个对象 作为CGoods的一个成员变量
				 // 也叫 成员对象
};

int main() {
	CGoods good("Cola", 2.5, 100, 2022, 8, 9);
	good.show();
	return 0;
}
```
类的对象，可以作为另一个类的成员变量，称为**成员对象**。

前面说过，一个对象的定义，需要两步（分配内存，调用构造函数），所以如果一个类中包含其他成员对象，实例化这个类的对象，分配内存时，会为成员对象分配内存，调用该类的构造函数时，需要调用成员对象的构造函数。

例如上面的代码，`CGoods`中有`CDate`成员对象 所以需要在`CGoods`的构造函数中，传入`CDate`的构造函数的参数，并调用`CDate`的构造函数，调用方法是在函数列表的`):`后调用。在`:`后的内容叫做**初始化列表**。

构造函数调用时，初始化列表中的内容先执行，然后执行构造函数函数体内的内容。

### 初始化顺序
类的成员变量的初始化顺序，和它们**定义的顺序**相同，和初始化列表中变量的顺序无关。

示例：  
```C++
#include <iostream>
using namespace std;
class CTest {
public:
	CTest(int data = 10) : b(data), a(b) {}
	void show() { cout << a << endl << b << endl; }
private:
	int a;
	int b;
};
int main() {
	CTest t;
	t.show(); // 输出为 0 \n 10
	return 0;
}
```
成员变量初始化顺序为：`a`，`b`。`a`用`b`的值初始化时，`b`还没有初始化，所以用栈上的随机值，`b`初始化为`data`的值。



## 类的各种成员及区别

### 普通成员方法
- 属于类的作用域；
- 必须由对象进行调用，原因是在编译阶段，成员方法的参数列表会添加一个指向该类对象的指针形参，调用该成员方法的对象地址作为实参入栈；
- 可以任意访问对象的私有成员变量；（先不考虑保护的成员变量）


### 静态成员变量
- 用`static`进行修饰；
- 在类内进行**声明**（`static int _count;`），在类外进行**定义和初始化**（`int Test::_count = 0;`）；
- 所有该类的对象共用一个静态变量，不占用对象的内存；（相当于一个落在类的作用域中的全局变量）
- 对静态成员变量的访问，最好使用静态成员方法。

示例代码：  
```C++
#include <iostream>
using namespace std;

class Test {
private:
	static int _count; // 声明  静态int类型用于记录初始化过的Test对象个数
	int _a;
public:
	Test(int a = 0) : _a(a) { ++ _count; }
	static int count() { return _count; }
	// int count() { return _count; }
	// 这种写法是不合适的 原因是
	// 普通成员方法 编译时会在参数列表中添加调用对象的指针
	// 但是 Test::count(); 并不需要特定对象进行调用 所以错误
	// 可以使用 test.count(); 方式调用
};
// 定义 静态成员变量 必须要在类外 进行定义并初始化
int Test::_count = 0;

int main()
{
	Test a, b, c;
	cout << Test::count() << endl;
	return 0;
}
```

### 静态成员方法
- 用`static`修饰，属于类的作用域；
- 调用静态成员方法，不需要依赖于对象，可以直接使用类名调用（`Test::count();`）；
- 静态成员方法，在编译时，参数列表中不会添加指向对象的`this`指针，这是它不需要依赖于对象的原因；（和普通成员方法的本质区别）
- 静态成员方法不能访问普通的成员变量，应该用于访问其他静态的成员变量（方法）。



### 常成员方法 - const
考虑下面的代码：  
```C++
#include <iostream>
using namespace std;

class Test {
private:
	static int _count;
	int _a;
public:
	Test(int a = 0) : _a(a) { ++ _count; }
	static int count() { return _count; }
	void show() const { cout << a << endl; } // 常对象只能调用常成员方法 由const修饰
	// void show() { cout << a << endl; }
	// 使用const对象调用这种方法就错误了
};
int Test::_count = 0;

int main()
{
	Test a, b, c;
	cout << Test::count() << endl;
	const Test t;
	t.show();
	return 0;
}
```

如果我们使用一个常对象，能否调用普通成员方法呢？  
答案是，不能。分析如下：

如果我们使用常对象（`const Test t;`）来调用普通成员方法（`t.show()`），实际上是这样调用的：`Test::show(&t)`，实参`&t`的类型为`const Test*`，而`Test::show()`的形参类型为`Test*`，用`Test*`的指针来接收`const Test*`的值是不允许的。

所以，常对象只能调用常成员方法（`void show() const;`），常成员方法，在编译时，形参列表中添加的指针类型为`const Test* this`，和常对象地址类型相同。同时，普通对象地址类型`Test*`可以被`const Test* this`接收，所以普通对象也可调用常成员方法。

结论：  
- 常对象只能调用常成员方法；
- 普通对象也可以调用常成员方法；
- 对对象成员不进行修改的方法，尽量都使用常成员方法。



## 指向类成员的指针

示例代码：  
```C++
#include <iostream>
using namespace std;

class Test {
public:
	void func() { cout << "Test::func" << endl; }
	static void static_func() { cout << "Test::static_func" << endl; }
	int a;
	static int b;
};
int Test::b;

int main()
{
	Test t1;
	Test* t2 = new Test();
// 指向成员变量的指针
	// int* p = &Test::a; 
	// 错误 int* 无法接收 int Test::*
	// 需要指定类作用域
	int Test::* p = &Test::a;
	
	// *p = 20;
	// 错误 对成员指针的解引用操作 必须依赖对象
	t1.*p = 20;
	t2->*p = 30;
	
	int* p1 = &Test::b; // 不需要指定类作用域
	*p1 = 40; // 静态成员变量不依赖对象
// 指向成员方法的指针
	// void(*pfunc)() = &Test::func;
	// 错误 指向成员方法的指针需要指定类作用域
	void(Test::*pfunc)() = &Test::func;
	
	// (*pfunc)();
	// 错误 普通成员方法 必须依赖对象
	(t1.*pfunc)();
	(t2->*pfunc)();
	
	void(*p1func)() = &Test::static_func; // 不需要指定类作用域
	(*p1func)(); // 不依赖对象
	delete t2;
	return 0;
}
```



