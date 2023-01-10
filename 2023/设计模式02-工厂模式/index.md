# 【设计模式】02 - 工厂模式


## 工厂模式

工厂模式，也是一种创建型的设计模式，主要是封装了对象的创建。

在程序中有很多类的情况下，我们在`new`对象时需要记住很多的类名，而且在类名更改时，该类创建对象的地方都需要修改，非常繁琐。可以使用工厂模式封装对象的创建过程，，无需我们自己`new`对象了。

工厂模式也分为几种类型：  
- 简单工厂(Simple Factory)
- 工厂方法(Factory Method)
- 抽象工厂(Abstract Factory)

## 问题场景

我们有一些不同品牌的汽车类，创建这些类的对象，需要我们自己`new`。

代码：  
```cpp
#include <iostream>

class Car
{
public:
    Car(std::string name) : name_(name) {}
    virtual void show() = 0;

protected:
    std::string name_;
};

class BMW : public Car
{
public:
    BMW(std::string name) : Car(name) {}
    void show()
    {
        std::cout << "This is a BMW" << std::endl;
    }
};

class Audi : public Car
{
public:
    Audi(std::string name) : Car(name) {}
    void show()
    {
        std::cout << "This is an Audi" << std::endl;
    }
};

int main(int argc, char **argv)
{
    Car *p1 = new BMW("X1");
    Car *p2 = new Audi("A6");
    p1->show();
    p2->show();
    delete p1;
    delete p2;
    return 0;
}
```

## 简单工厂(Simple Factory)

如果车的型号很多，使用`new`不方便，就可以用简单工厂模式来封装各种类的对象创建。

缺点：  
- 将所有的产品(类)放在一个工厂中生产，不符合逻辑，在这个例子中，应该是BMW车在一个工厂中生产，Audi车在一个工厂中生产。
- 无法做到软件设计的**开-闭原则**，增加、删除类都需要对工厂生产函数进行修改。

代码：  
```cpp
#include <iostream>
#include <memory>

class Car
{
public:
    Car(std::string name) : name_(name) {}
    virtual void show() = 0;

protected:
    std::string name_;
};

class BMW : public Car
{
public:
    BMW(std::string name) : Car(name) {}
    void show()
    {
        std::cout << "This is a BMW" << std::endl;
    }
};

class Audi : public Car
{
public:
    Audi(std::string name) : Car(name) {}
    void show()
    {
        std::cout << "This is an Audi" << std::endl;
    }
};

enum CarType
{
    bmw,
    audi
};

class SimpleFactory
{
public:
    Car *createCar(CarType ct)
    {
        switch (ct)
        {
        case bmw:
            return new BMW("X1");
        case audi:
            return new Audi("A6");
        default:
            std::cerr << "wrong cartype: " << ct << std::endl;
            break;
        }
        return nullptr;
    }
};

int main(int argc, char **argv)
{
    std::unique_ptr<SimpleFactory> factory(new SimpleFactory());
    std::unique_ptr<Car> p1(factory->createCar(bmw));
    std::unique_ptr<Car> p2(factory->createCar(audi));
    p1->show();
    p2->show();
    return 0;
}
```

## 工厂方法(Factory Method)

使用工厂抽象基类提供接口，不同车的工厂继承自抽象基类，解决了不同车型在同一个工厂生产的逻辑问题，且满足了**开-闭原则**，添加、删除一个工厂不需要修改基类。

代码：  
```cpp
#include <iostream>
#include <memory>

class Car
{
public:
    Car(std::string name) : name_(name) {}
    virtual void show() = 0;

protected:
    std::string name_;
};

class BMW : public Car
{
public:
    BMW(std::string name) : Car(name) {}
    void show()
    {
        std::cout << "This is a BMW" << std::endl;
    }
};

class Audi : public Car
{
public:
    Audi(std::string name) : Car(name) {}
    void show()
    {
        std::cout << "This is an Audi" << std::endl;
    }
};

// Factory Method
class Factory
{
public:
    virtual Car *createCar(std::string name) = 0;
};

// BMW Factory
class BMWFactory : public Factory
{
public:
    Car *createCar(std::string name)
    {
        return new BMW(name);
    }
};

// Audi Factory
class AudiFactory : public Factory
{
public:
    Car *createCar(std::string name)
    {
        return new Audi(name);
    }
};

// ... other Factory

int main(int argc, char **argv)
{
    std::unique_ptr<Factory> BMWfactory(new BMWFactory());
    std::unique_ptr<Factory> Audifactory(new AudiFactory());
    std::unique_ptr<Car> p1(BMWfactory->createCar("X6"));
    std::unique_ptr<Car> p2(Audifactory->createCar("A8"));
    p1->show();
    p2->show();
    return 0;
}
```

## 抽象工厂(Abstract Factory)
