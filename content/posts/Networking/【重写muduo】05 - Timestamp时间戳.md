---
title: 【重写muduo】05 - Timestamp时间戳
date: 2022-09-15
tags: [network, C++, muduo]
categories: [Networking]
---

时间戳类，我们只实现时间戳的核心功能，返回当前时间戳以及时间戳转格式化字符串。

使用c++11标准的`chrono`库来获取微秒级时间戳，使用c库函数`localtime`来获取格式化时间`struct std::tm`，进行字符串输出。


## 源码

`Timestamp.hh`  
```cpp
#ifndef   __TIMESTAMP_HH_
#define   __TIMESTAMP_HH_

#include <iostream>
#include <string>


/* 时间戳类 */
class Timestamp {
public:
    /* 构造 */
    Timestamp();
    explicit Timestamp(int64_t microSecondsSinceEpoch);
    /* 获得当前时间戳 */
    static Timestamp now();
    /* 时间戳转字符串 */
    std::string toString() const;
private:
    /* 微秒数时间戳 */
    int64_t microSecondsSinceEpoch_;
};


#endif // __TIMESTAMP_HH_
```


`Timestamp.cc`  
```cpp
#include "Timestamp.hh"

#include <chrono>
    
/* 构造 */
Timestamp::Timestamp() : microSecondsSinceEpoch_(0) {}
Timestamp::Timestamp(int64_t microSecondsSinceEpoch) 
    : microSecondsSinceEpoch_(microSecondsSinceEpoch) {}

/* 获得当前时间戳 从1970.01.01:00:00:00开始到现在的微秒数 */
Timestamp Timestamp::now() {
    /* 使用c++11 chrono库来获取时间戳 */
    using namespace std::chrono;
    auto time_now = system_clock::now();
    auto duration_in_us = duration_cast<microseconds>(time_now.time_since_epoch());
    return Timestamp(duration_in_us.count());
}

/* 时间戳转字符串 */
std::string Timestamp::toString() const {
    char buf[128] = {0};
    int64_t seconds = microSecondsSinceEpoch_ / 1000000;
    int64_t microseconds = microSecondsSinceEpoch_ % 1000000;
    time_t seconds_since_epoch = static_cast<time_t>(seconds);
    std::tm tm_time = *std::localtime(&seconds_since_epoch);
    /* 格式化输出时间戳 yyyy/mm/dd hh:mm:ss s.us*/
    snprintf(buf, sizeof(buf), "%4d/%02d/%02d %02d:%02d:%02d %jd.%06jd", 
        tm_time.tm_year + 1900,
        tm_time.tm_mon + 1,
        tm_time.tm_mday,
        tm_time.tm_hour,
        tm_time.tm_min,
        tm_time.tm_sec,
        seconds,
        microseconds);
    return buf;
}
```