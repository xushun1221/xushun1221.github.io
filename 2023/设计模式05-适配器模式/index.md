# 【设计模式】05 - 适配器模式


## 适配器模式 Adapter

适配器模式也是一种结构型模式，主要的功能是让不兼容的接口能够在一起工作。

> 在项目开发中往往会使用到一些第三方的代码库，它们的接口设计很可能和项目代码本身的接口设计不兼容，这就需要使用适配器模式来为第三方库包装一层接口。这也是适配器模式较常用的场景。



问题场景：有一个电脑类`Computer`只支持通过VGA输出播放，有一个`TV01`类只能通过VGA输入播放，有一个`TV02`类只能通过HDMI接口播放，电脑可以直接连接VGA接口的电视，但是连接HDMI接口的电视则需要适配器转接。

代码：  
```cpp
#include <iostream>

// VGA接口
class VGA
{
public:
    virtual void play() = 0;
};
// 使用VGA接口输入的TV
class TV01 : public VGA
{
public:
    void play() { std::cout << "通过VGA接口连接播放" << std::endl; }
};

// HDMI接口
class HDMI
{
public:
    virtual void play() = 0;
};
// 使用HDMI接口输入的TV
class TV02 : public HDMI
{
public:
    void play() { std::cout << "通过HDMI接口播放" << std::endl; }
};

// VGA到HDMI接口的适配器
class VGA2HDMIAdapter : public VGA // 从外部看是一个VGA接口
{
public:
    VGA2HDMIAdapter(HDMI *p) : pHDMI(p) {}
    void play() { pHDMI->play(); } // 接口转换
private:
    HDMI *pHDMI; // 内部实际上是HDMI接口在工作
};

// 只支持VGA接口输出的电脑
class VGAComputer
{
public:
    void playVideo(VGA *pVGA) { pVGA->play(); }
};

int main(int argc, char **argv)
{
    // VGA接口的TV和VGA接口的电脑直接连接
    TV01 *tv1 = new TV01();
    VGAComputer *p1 = new VGAComputer();
    p1->playVideo(tv1);

    TV02 *tv2 = new TV02();
    // p1->playVideo(tv2); 接口不兼容
    // 使用适配器将VGA转为HDMI
    p1->playVideo(new VGA2HDMIAdapter(tv2));
    return 0;
}
```
