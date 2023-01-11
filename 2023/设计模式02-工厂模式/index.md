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

如果车的型号很多，使用`new`不方便，就可以用简单工厂模式来封装各种类的对象创建。把对象的创建封装在一个接口函数里面，通过传入不同的标识，返回不同的对象。

优点：用户不用自己`new`对象，不用了解对象创建的具体过程。

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

优点：  
- 不同的产品在不同的工厂中生产
- 符合开-闭原则。

缺点：  
- 很多产品之间是有关联的，属于一个产品簇，逻辑上应该在同一个工厂中生产
- 每个产品对应一个工厂，工厂太多，不好维护

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

优点：抽象工厂可以提供一组有关联关系的产品组的对象的统一创建。

缺点：如果某个工厂添加了一项产品，那么就需要在抽象基类中添加这个方法，那么其他的工厂也必须添加这个方法的实现。

实现方法是，在抽象工厂的抽象基类中，提供多种产品的创建接口，具体的各个工厂类来实现这些接口，这样就可以在一个工厂中创建多个产品对象了。

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

// other products
class CarLight
{
public:
    virtual void show() = 0;
};
class BMWLight : public CarLight
{
public:
    void show()
    {
        std::cout << "BMW Light" << std::endl;
    }
};
class AudiLight : public CarLight
{
public:
    void show()
    {
        std::cout << "Audi Light" << std::endl;
    }
};

class AbstractFactory
{
public:
    virtual Car *createCar(std::string name) = 0;
    virtual CarLight *createCarLight() = 0;
};

class BMWFactory : public AbstractFactory
{
public:
    Car *createCar(std::string name)
    {
        return new BMW(name);
    }
    CarLight *createCarLight()
    {
        return new BMWLight();
    }
};

class AudiFactory : public AbstractFactory
{
public:
    Car *createCar(std::string name)
    {
        return new Audi(name);
    }
    CarLight *createCarLight()
    {
        return new AudiLight();
    }
};

int main(int argc, char **argv)
{
    std::unique_ptr<AbstractFactory> BMWfactory(new BMWFactory());
    std::unique_ptr<AbstractFactory> Audifactory(new AudiFactory());
    std::unique_ptr<Car> p1(BMWfactory->createCar("X6"));
    std::unique_ptr<Car> p2(Audifactory->createCar("A8"));
    std::unique_ptr<CarLight> l1(BMWfactory->createCarLight());
    std::unique_ptr<CarLight> l2(Audifactory->createCarLight());
    p1->show();
    p2->show();
    l1->show();
    l2->show();
    return 0;
}
```
