# 【设计模式】03 - 代理模式


代理模式、装饰器模式、适配器模式等，不同于创建型的模式，它们属于**结构型模式**，主要关注类和对象的组合，功能的使用。

## 代理模式

代理模式是这样一种模式，功能由委托类来实现，而代理类的作用是控制委托类对象的访问权限。

举个例子，有个电影网站，电影有三种类型，免费的、vip的、需要观影券的，观看这三种电影的功能函数由委托类实现，而对于委托类的访问权限则由代理类控制。如果用户是一个游客，那么他通过游客的代理类来访问委托类的功能，他只能调用免费电影观看功能。如果他是vip，他就通过vip的代理类来访问委托类的功能，他可以看免费电影和vip电影。

代码：  
```cpp
#include <iostream>
#include <memory>

// 抽象类
class VideoSite
{
public:
    virtual void freeMovie() = 0;   // 观看免费电影
    virtual void vipMovie() = 0;    // 观看vip电影
    virtual void ticketMovie() = 0; // 观看需要观影券的电影
};

// 委托类 实现功能
class FixBugVideoSite
{
public:
    virtual void freeMovie() { std::cout << "观看免费电影" << std::endl; }
    virtual void vipMovie() { std::cout << "观看vip电影" << std::endl; }
    virtual void ticketMovie() { std::cout << "观看需要观影券的电影" << std::endl; }
};

// 代理类 游客通过该类访问freeMovie()功能
class FreeVideoSiteProxy : public VideoSite
{
public:
    virtual void freeMovie() { pVideo.freeMovie(); } // 只能看免费电影
    virtual void vipMovie() { std::cout << "您不是vip用户 无法观看vip电影" << std::endl; }
    virtual void ticketMovie() { std::cout << "您没有观影券 无法观看该电影" << std::endl; }

private:
    // 组合委托类 通过委托类访问观影的功能函数
    FixBugVideoSite pVideo;
};

// 代理类 vip通过该类访问freeMovie()和vipMovie()功能
class VipVideoSiteProxy : public VideoSite
{
public:
    virtual void freeMovie() { pVideo.freeMovie(); }
    virtual void vipMovie() { pVideo.vipMovie(); }
    virtual void ticketMovie() { std::cout << "您没有观影券 无法观看该电影" << std::endl; }

private:
    FixBugVideoSite pVideo;
};

int main(int argc, char **argv)
{
    // 游客和vip用户使用不同的代理类来访问观影功能
    std::unique_ptr<VideoSite> p1(new FreeVideoSiteProxy()); // 游客代理
    std::unique_ptr<VideoSite> p2(new VipVideoSiteProxy());  // vip代理

    p1->freeMovie();
    p1->vipMovie();
    p1->ticketMovie();
    
    p2->freeMovie();
    p2->vipMovie();
    p2->ticketMovie();
}
```
