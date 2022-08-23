# 【C++基础】06 - 继承与多态


## 继承的本质和原理

继承的本质：代码的复用

类和类之间的关系：  
1. 组合：a part of
2. 继承：a kind of

三种继承方式下，派生类对基类成员的访问限定情况：

|继承方式|基类的访问限定|派生类的访问限定(Y/N)|外部的访问限定(Y/N)|
|---|---|---|---|
|public|public|public(Y)|Y|
||protected|protected(Y)|N|
||private|不可见的(N)|N|
|protected|public|protected(Y)|N|
||protected|protected(Y)|N|
||private|不可见的(N)|N|
|private|public|private(Y)|N|
||protected|private(Y)|N|
||private|不可见的(N)|N|

- 公有继承（public）下
  - 基类的public和protected成员在派生类中还是public和protected成员；
  - 基类的private成员在派生类中不可见。
- 保护继承（protected）下
  - 基类的public和protected成员在派生类中都为protected；
  - 基类的private成员在派生类中不可见。
- 私有继承（private）下
  - 基类public和protected成员在派生类中都为private；
  - 基类的private成员在派生类中不可见。

总结：  
- 派生类对基类成员的访问权限，不会超过继承方式；
- 外部只能访问对象的public成员；
- 继承结构中，派生类可以从基类继承来private成员，但是不可直接访问；
- protected和private成员的区别？
  - 在基类中定义的成员，想被派生类访问，但不想被外部访问，就在基类中将其限定为protected；
  - 如果不想成员被派生类和外部访问，就在基类中将其定义为private。

其他注意点：  
- 如果某个类继承自某个派生类，它的访问限定情况由它的直接基类决定；
- 默认的继承方式，由派生类的定义方式决定
  - 使用class定义，默认继承方式为private（class定义类，内部成员的默认限定符也是private）
  - 使用struct定义，默认继承方式为public（struct定义类，内部成员的默认限定符也是public）

## 派生类的构造、析构过程

派生类可以从基类继承所有的成员（变量和方法），除了构造函数和析构函数。

- 派生类的构造和析构函数，负责初始化和清理派生类部分的成员；
- 派生类从基类继承来的成员，由基类的构造和析构函数负责；
- 派生类通过调用基类的构造函数来初始化基类部分的成员；
- 基类的析构函数无需手动调用。

派生类对象的构造和析构过程如下：  
1. 派生类调用基类的构造函数，初始化从基类继承来的成员；
2. 调用派生类自己的构造函数，初始化自己特有的成员；
3. ...派生类对象作用域到期...
4. 调用派生类的析构函数，释放派生类成员占用的资源；
5. 调用基类的析构函数，释放派生类内存中，从基类继承来的成员占用的资源。

示例：　　
```C++
#include <iostream>
class Base {
public:
    Base(int data) : a(data) { std::cout << "Base() &a:" << &a << std::endl; }
    ~Base() { std::cout << "~Base()" << std::endl; }
protected:
    int a;
};
class Derive : public Base {
public:
    // Derive(int data) : a(data), b(data) {} 错误写法 不能在构造函数中给基类成员初始化
    Derive(int data) : Base(data), b(data) { std::cout << "Derive() &b:" << &b << std::endl; }
    ~Derive() { std::cout << "~Derive()" << std::endl; }
private:
    int b;
};
int main() {
    Derive test(10);
    return 0;
}
/* 输出
xushun@xushun-virtual-machine:~/cppmiddle$ ./a.out 
Base() &a:0x7ffcf2a82310
Derive() &b:0x7ffcf2a82314
~Derive()
~Base()
*/
```


## 重载、隐藏、覆盖

这一节讲重载和隐藏，覆盖的内容在下一节虚函数中。

### 重载和隐藏
先说结论：  
- 重载关系：一组函数重载的前提是，它们在**同一个作用域中**，且**函数名相同**，参数列表不同。这里的作用域说的是在同一个类中。
- 隐藏关系（作用域的隐藏）：在继承结构中，派生类的成员，会把基类的同名成员隐藏。

示例：  
```C++
#include <iostream>
class Base {
public:
    Base(int data = 10) : a(data) {}
    void show() { std::cout << "Base::show()" << std::endl; }
    void show(int) { std::cout << "Base::show(int)" << std::endl; }
protected:
    int a;
};
class Derive : public Base {
public:
    Derive(int data = 10) : Base(data), b(data) {}
    void show() { std::cout << "Derive::show()" << std::endl; } // #1
    void show(int) { std::cout << "Derive::show(int)" << std::endl; } // #2
private:
    int b;
};

int main() {
    Derive test;
    test.show();
    test.show(0);
    test.Base::show();
    test.Base::show(0);
    return 0;
}
```
观察上面的代码。

存在两个重载关系：  
1. `Base::show()`和`Base::show(int)`是重载关系，它们同属于`Base::`，函数名相同，参数列表不同；
2. `Derive::show()`和`Derive::show(int)`是重载关系，它们同属于`Derive::`，函数名相同，参数列表不同；

注释掉`#1#2`两行，输出为：  
```console
Base::show()
Base::show(int)
Base::show()
Base::show(int)
```
`Derive::`中没有`show()`和`show(int)`，所以从`Base::`中寻找。

只注释掉`#2`，无法通过编译，输出为：  
```console
inherit.cpp: In function ‘int main()’:
inherit.cpp:22:16: error: no matching function for call to ‘Derive::show(int)’
     test.show(0);
                ^
inherit.cpp:13:10: note: candidate: ‘void Derive::show()’
     void show() { std::cout << "Derive::show()" << std::endl; } // #1
          ^~~~
inherit.cpp:13:10: note:   candidate expects 0 arguments, 1 provided
```
错误提示说明，`Derive::show()`隐藏掉了`Base::show()`和`Base::show(int)`，所以`test.show(int)`时不会在`Base::`中寻找，同时注释掉了`#2`，所以找不到该函数的定义。如果只注释掉`#1`，情况相同，`Derive::show(int)`会隐藏`Derive::`中同名的两个函数（参数列表不同也会被隐藏）。

不注释代码，输出：  
```console
Derive::show()
Derive::show(int)
Base::show()
Base::show(int)
```
不指定作用域的话，优先使用派生类作用域中的成员，隐藏了基类的成员，如果在派生类中没有，才会取寻找基类中继承的成员。如果要使用基类被派生类隐藏的成员，需要使用作用域运算符`Base::`指定。

### 基类和派生类的转换、指针（引用）
> 我们把继承结构说成从上（基类）到下（派生类）的结构。

结论：  
在继承结构中进行上下类型的转换（基类 <->派生类），默认只支持从下到上的转换（基类 <- 派生类）

示例：  
```C++
#include <iostream>
class Base {
public:
    Base(int data = 10) : a(data) {}
protected:
    int a;
};
class Derive : public Base {
public:
    Derive(int data = 10) : Base(data), b(data) {}
private:
    int b;
};

int main() {
    Base b;
    Derive d;
    // 基类对象 <- 派生类对象  从下到上 Y
    b = d;
    // 派生类对象 <- 基类对象  从上到下 N
    //d = b;
    // 基类指针（引用） <- 派生类指针  从下到上 Y
    Base* pb = &d;
    // 派生类指针（引用） <- 基类指针  从下到上 N
    //Derive* pd = &b;
    // 派生类指针不能指向一个基类对象 
    // 因为派生类的内存空间大于基类 派生类指针指向基类对象
    // 会造成内存的非法访问
    Derive* pd = (Derive*)&b; // 这样很危险!
    return 0;
}
```



## 虚函数、静态绑定、动态绑定、覆盖

静态绑定是**编译时期**的函数调用，调用的是普通函数。  
动态绑定是**运行时期**的函数调用，且调用的一定是**虚函数**。

### 静态绑定

示例1：  
```C++
#include <iostream>
#include <typeinfo>
class Base {
public:
    Base(int data = 10) : a(data) {}
    void show() { std::cout << "Base::show()" << std::endl; }
    void show(int) { std::cout << "Base::show(int)" << std::endl; }
protected:
    int a;
};
class Derive : public Base {
public:
    Derive(int data = 10) : Base(data), b(data) {}
    void show() { std::cout << "Derive::show()" << std::endl; }
private:
    int b;
};
int main() {
    Derive d(50);
    Base* pb = &d;
    // 静态（编译阶段）绑定（函数调用）
    pb->show(); // 静态绑定 call Base::show()
    pb->show(0); // 静态绑定 call Base::show(int)
    std::cout << sizeof(Base) << std::endl;
    std::cout << sizeof(Derive) << std::endl;
    std::cout << typeid(pb).name() << std::endl;
    std::cout << typeid(*pb).name() << std::endl;
    return 0;
}
/* 输出
xushun@xushun-virtual-machine:~/cppmiddle$ ./a.out 
Base::show()
Base::show(int)
4
8
P4Base
4Base
*/
```
在上面的代码中，我们使用了`Base*`指针指向一个`Derive`对象，用指针调用函数，编译器就会在`Base::`作用域中寻找函数，如果发现不是虚函数，那么直接调用它（`call`）。这也称**静态绑定**，是在编译阶段进行的函数调用。

再看内存大小，`Base`对象中只有一个`int`成员变量，所以为4字节，`Derive`对象中有从基类继承来的一个`int`和自己的一个`int`，所以8字节。

再看类型，虽然`Base*`类型指针指向的是`Derive`对象，但`*pb`仍被编译器认定为`Base`类型。

### 虚函数、动态绑定、覆盖

示例2：  
```C++
#include <iostream>
#include <typeinfo>
class Base {
public:
    Base(int data = 10) : a(data) {}
    // 定义为虚函数
    virtual void show() { std::cout << "Base::show()" << std::endl; }
    virtual void show(int) { std::cout << "Base::show(int)" << std::endl; }
protected:
    int a;
};
class Derive : public Base {
public:
    Derive(int data = 10) : Base(data), b(data) {}
    // 返回值、函数名、参数列表 都和基类中的虚函数相同
    // 自动定义为虚函数
    void show() { std::cout << "Derive::show()" << std::endl; } // 这是一个虚函数 虽然没用virtual修饰
    // 它覆盖了 Base::show()
private:
    int b;
};
int main() {
    Derive d(50);
    Base* pb = &d;
    pb->show(); // 动态（运行阶段）绑定
    /* 动态绑定 对应的汇编如下
        mov eax, dword ptr[pb]  // 将 d 虚函数指针 放入 eax
        mov ecx, dword ptr[eax] // 将 虚函数指针 指向的函数地址Derive::show() 放入 ecx
        call ecx // 调用虚函数
    */
    pb->show(0); // 动态绑定
    std::cout << sizeof(Base) << std::endl;
    std::cout << sizeof(Derive) << std::endl;
    std::cout << typeid(pb).name() << std::endl;  // pb 还是 Base*
    std::cout << typeid(*pb).name() << std::endl; // *pb 就变为 Derive
    // 如果Base没有虚函数 *pb 就会被识别为编译时期的类型 就是 Base
    // 如果Base有虚函数 *pb 就会被识别为运行时期的类型 就是 RTTI类型 Derive
    return 0;
}
/* 输出
xushun@xushun-virtual-machine:~/cppmiddle$ ./a.out 
Derive::show()
Base::show(int)
16
16
P4Base
6Derive
*/
```

#### vftable虚函数表、RTTI、vptr虚函数指针

当类中定义了虚函数，那么在**编译阶段**，编译器会为这个类类型产生一个**唯一**的**vftable虚函数表**，虚函数表中主要存储的内容是**RTTI指针（Run-Time Type information）**和**虚函数的地址**，RTTI指针可以理解为一个指向类型字符串常量的指针。当程序运行时，每一张虚函数表都会加载到内存的`.rodata`只读数据段中。

`Base`类的vftable虚函数表：（可以使用`g++ -fdump-lang-class virtual.cpp`命令查看类的内存布局）

|地址偏移|内容|
|---|---|
|0|0|
|8|`&RTTI` (指向`"Base"`)|
|16|`&Base::show()` (虚函数的地址)|
|24|`&Base::show(int)`|


当类中定义了虚函数，那么由该类实例化的对象，其内存起始位置会多存储一个**vptr虚函数指针**，指向该类对应的vftable虚函数表的虚函数地址起始位置，例如`Base`类对象的虚函数指针，指向`Base`类虚函数表偏移16位置。多个对象的虚函数指针，指向**同一个**虚函数表。

一个`Base`对象的内存结构：

|地址偏移|内容|
|---|---|
|0|`vptr`（指向`Base`虚函数表偏移16位置）|
|8|`int a`|
|12-16|内存对齐|

这里也可以看出，一个类的虚函数的个数，不影响实例化对象的内存大小，影响的是虚函数表的大小。

#### 虚函数的覆盖

如果基类有虚函数，派生类当然会继承基类的虚函数，那么也有虚函数表。如果一个派生类中的方法，和基类继承来的某个方法，**返回值、函数名、参数列表都相同**，而且基类的方法是virtual虚函数，那么派生类的这个方法，编译器会自动处理为虚函数（编译阶段），并且会将派生类的虚函数表中，原来的基类的虚函数，替换为派生类的对应虚函数。这个动作就叫做虚函数的**覆盖**（重写）。

`Derive`类的虚函数表：

|地址偏移|内容|
|---|---|
|0|0|
|8|`&RTTI`（指向`"Derive"`）|
|16|`&Derive::show()` (`&Base::show()`被覆盖了)|
|24|`&Base::show(int)` (没有重写)|


一个`Derive`对象的内存结构：

|地址偏移|内容|
|---|---|
|0|`vptr`（指向`Derive`虚函数表偏移16位置）|
|8|`int Base::a`|
|12|`int Derive::b`|


#### 动态绑定

参考上面的示例2，当编译到`pb->show();`时，因为`pb`是`Base*`类型的指针，所以进入`Base::`作用域查看，发现`show()`是virtual虚函数，就进行**动态绑定**。过程是：通过`pb`指向的`Derive`对象`d`的虚函数指针`vptr`找到，`Derive`的虚函数表，获得`show()`虚函数的地址（因为`Base::show()`被`Derive::show()`覆盖，所以得到了`Derive::show()`的地址），然后`call`这个地址。

当编译到`pb->show(0);`时，进入`Base::`作用域查看，发现`show(int)`是虚函数，进行动态绑定，在`Derive`虚函数表中找到`show(int)`的地址（没有重写这个函数，所以还是得到了`Base::show()`的地址），`call`之。

动态的含义是，**运行时**进行动态绑定，只有在程序运行时，才会通过虚函数表找到函数，而静态绑定，是在**编译时**就能确定调用的函数。


> 为什么动态绑定要在运行时确定要调用的函数呢？  
> 答：因为，调用虚函数的基类指针，不一定在编译时就能确定指向的对象类型。需要通过对象的vptr虚函数指针，确定对象的类型，并调用正确的虚函数。  
> 例如下面的代码，在编译时期，你能看出`pb`调用的是哪个函数吗？  
> ```C++
> Base* pb = nullptr;
> std::string type;
> std::cin >> type;
> if (type == "Base") {
>   pb = new Base();
> } else {
>   pb = new Derive();
> }
> pb->show();
> ```

其实**绑定**就是确定对象指针应该调用的函数的过程。调用的函数是普通函数，就进行静态绑定，而调用虚函数就进行动态绑定。


## 虚析构函数

### 哪些函数不能实现为虚函数？

首先明确一个概念，虚函数依赖于什么？  
1. 虚函数要能产生地址，并存储在vftable虚函数表中；
2. 对象必须存在，因为实例化的对象才有vptr虚函数指针指向虚函数表，才能找到虚函数地址。

构造函数，**不能**实现为虚函数：  
1. 构造函数执行完成后，对象才产生，所以不能加`virtual`；
2. 在构造函数中调用的函数（哪怕它是虚函数），都是**静态绑定**。

static静态成员方法，**不能**实现为虚函数。因为static静态方法不依赖于对象。

### 析构函数可以实现为虚函数

析构函数调用的时候，对象是存在的。

什么时候应该使用虚析构函数呢？看下面的代码：  
```C++
#include <iostream>
class Base {
public:
    Base(int data = 10) : a(data) { std::cout << "Base()" << std::endl; }
    ~Base() { std::cout << "~Base()" << std::endl; }
    virtual void show() { std::cout << "Base::show()" << std::endl; }
protected:
    int a;
};
class Derive : public Base {
public:
    Derive(int data = 10) : Base(data), b(data) { std::cout << "Derive()" << std::endl; }
    ~Derive() { std::cout << "~Derive()" << std::endl; }
    void show() { std::cout << "Derive::show()" << std::endl; }
private:
    int b;
};
int main() {
    Base* pb = new Derive(10);
    pb->show();
    delete pb;
    return 0;
}
/* 输出
xushun@xushun-virtual-machine:~/cppmiddle$ ./a.out 
Base()
Derive()
Derive::show()
~Base()
*/
```
从输出中可以看出，对象构造时，`Base()`和`Derive()`都正确调用了，`pb->show()`也正确调用了覆盖的虚函数`Derive::show()`，但是对象析构时仅调用了基类的析构函数`~Base()`，在此之前应该调用`~Derive()`但是并没有，这就会造成内存泄漏。

`delete pb`只调用了`~Base()`，这个过程是：`pb`是`Base*`类型的指针，编译器进入`Base::`作用域查看，发现`~Base()`是普通函数，所以直接进行静态绑定（`call Base::~Base()`），`~Derive()`没有机会被调用。

如何解决这个问题？将基类的虚构函数定义为虚函数即可：  
```C++
#include <iostream>
class Base {
public:
    Base(int data = 10) : a(data) { std::cout << "Base()" << std::endl; }
    // 虚析构函数
    virtual ~Base() { std::cout << "~Base()" << std::endl; }
    virtual void show() { std::cout << "Base::show()" << std::endl; }
protected:
    int a;
};
class Derive : public Base {
public:
    Derive(int data = 10) : Base(data), b(data) { std::cout << "Derive()" << std::endl; }
    // 虽然函数名不同 但由于基类析构是虚函数 派生类析构自动成为虚函数
    ~Derive() { std::cout << "~Derive()" << std::endl; }
    void show() { std::cout << "Derive::show()" << std::endl; }
private:
    int b;
};
int main() {
    Base* pb = new Derive(10);
    pb->show();
    delete pb;
    return 0;
}
/* 输出
xushun@xushun-virtual-machine:~/cppmiddle$ ./a.out 
Base()
Derive()
Derive::show()
~Derive()
~Base()
*/
```
这样就可以正确调用派生类和基类的析构了。

原理是：`delete pb`时，调用`pb`指向的析构，`pb`是`Base*`，所以编译器进入`Base::`作用域查看，发现`~Base()`析构为虚函数，此时就需要进行**动态绑定**，由于`pb`指向一个`Derive`类型的对象，所以进入`Derive`的虚函数表查看析构，由于`Derive`类同样给出了析构（虽然析构的函数名不同，但是派生类的析构仍然会覆盖基类的虚析构函数），所以调用`Derive::~Derive()`析构。派生类的析构执行完成后，会自动调用基类的析构。

### 何时使用虚析构函数？

基类对象的指针（引用）指向**堆上**`new`出来的派生类对象时，需要使用虚析构函数，因为`delete`基类指针时，调用析构函数必须发生动态绑定，否则会导致派生类的析构无法正确调用。


## 再谈动态绑定 - 何时进行动态绑定？

问题：是否虚函数的调用一定是动态绑定？

肯定不是！前面说过，在构造函数中调用的函数，无论是否为虚函数，都不会进行动态绑定，原因是动态绑定需要对象的虚函数指针vptr，而构造函数执行完成后对象才被构造。

还有一种情况是，使用对象本身（而非对象的指针）调用虚函数，无需进行动态绑定。

使用对象指针调用虚函数，一定会进行动态绑定，无论基类对象指针指向基类对象还是派生类对象，都进行动态绑定。

事实上，动态绑定必须由对象指针（或引用变量）调用虚函数。指针指向哪个对象，就通过虚函数指针，调用该对象类型的同名覆盖虚函数，这样的动态绑定才是有意义的。

示例：  
```C++
#include <iostream>
class Base {
public:
    Base(int data = 10) : a(data) {}
    virtual void show() { std::cout << "Base::show()" << std::endl; }
protected:
    int a;
};
class Derive : public Base {
public:
    Derive(int data = 10) : Base(data), b(data) {}
    void show() { std::cout << "Derive::show()" << std::endl; }
private:
    int b;
};
int main() {
    Base b(10);
    Derive d(20);
    // 静态绑定 对象本身调用虚函数 查看汇编代码确定为静态绑定
    b.show();
    d.show();
    // 动态绑定
    Base* pb1 = &b;
    pb1->show();
    Base* pb2 = &d;
    pb2->show();
    // 动态绑定
    Base& rb1 = b;
    rb1.show();
    Base& rb2 = d;
    rb2.show();
    // 动态绑定
    Derive* pd1 = &d;
    pd1->show();
    Derive& rd1 = d;
    rd1.show();
    // 动态绑定
    Derive* pd2 = (Derive*)&b; // 危险操作
    pd2->show(); // pd2 -> b.vptr -> Base::vftable -> Base::show()
    return 0;
}
```

## 多态到底是什么？

- 静态多态（编译时期）：函数重载、模板（函数模板、类模板）
- 动态多态（运行时期）：在继承结构中，基类指针（引用）指向派生类对象，通过该指针调用同名覆盖方法（虚函数），基类指针会调用指向的派生类对象的覆盖方法，称为多态。多态是通过动态绑定实现的。


继承的好处是什么？  
1. 代码复用；
2. 在基类中提供统一的虚函数接口，在派生类中进行重写，然后就可以使用多态了。


## 抽象类


