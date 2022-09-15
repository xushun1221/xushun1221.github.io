# 【重写muduo】03 - noncopyable接口类


## 理解源码

在原muduo库中(`noncopyable.h`)有这样一个类：  
```cpp
class noncopyable
{
 public:
  noncopyable(const noncopyable&) = delete;
  void operator=(const noncopyable&) = delete;

 protected:
  noncopyable() = default;
  ~noncopyable() = default;
};
```

这个类的作用和和它的名字一样：不可复制。

它的**拷贝构造函数**和**赋值运算符重载函数**被禁用了，这里的作用是禁止对该类和它的派生类对象进行拷贝构造和赋值操作。因为在对它的派生类对象进行拷贝构造和赋值时，需要调用基类的对应方法，但是它们已经被禁用了。

它的**构造函数**和**析构函数**被声明为`protected`，这里的作用是，不允许用户直接构造一个`noncopyable`类的对象，而希望用户只能构造该类的派生类对象。因为声明为`protected`的方法，就无法被外部调用了，而派生类对象可以调用，这就让用户可以构造派生类对象，而无法直接构造`noncopyable`类的对象。


如果想要定义一个**不允许进行拷贝构造和赋值的类**，可以直接令其继承自`noncopyable`类。

如muduo库中的`TcpServer`类：  
```cpp
class TcpServer : noncopyable
{
  /*
    ...
  */
}
```
注意，派生类为`class`时，默认继承方式为`private`，基类中`protected`限定的内容在派生类中变为`private`，仍然可在派生类内部访问。


## 我的实现

`noncopyable.hh`  
```cpp
#ifndef   __NONCOPYABLE_HH_
#define   __NONCOPYABLE_HH_

/* 接口类 禁用拷贝构造和赋值 */
class noncopyable {
public:
    noncopyable(const noncopyable&) = delete;
    void operator=(const noncopyable&) = delete;
protected:
    noncopyable() = default;
    ~noncopyable() = default;
};

#endif // __NONCOPYABLE_HH_
```
