---
title: 【重写muduo】04 - Logger日志系统
date: 2022-09-15
tags: [network, C++, muduo]
categories: [Networking]
---

日志系统，我们使用和muduo库不同的实现，日志直接输出到控制台。


## 单例模式

我们使用单例模式来实现`Logger`类。

使用`static Logger& instance();`接口来获得唯一的`Logger`实例对象。

## 日志等级

对于不同类型的日志，我们将其分为四级：  
1. `INFO`：普通信息；
2. `ERROR`：错误信息；
3. `FATAL`：core信息；
4. `DEBUG`：调试信息。

## 日志格式

`[log level]Timestamp : msg`


## 使用接口

通过四个宏函数分别输出不同日志等级的日志。

对于`DEBUG`级别的日志，通过一个宏开关`MUDEBUG`来限定其必须在调试模式下使用。



## 源码

`Logger.hh`  
```cpp
#ifndef   __LOGGER_HH_
#define   __LOGGER_HH_

#include <string>

#include "noncopyable.hh"

/* 定义四个宏 对应使用四个级别的日志 */

/* LOG_INFO("%s %d", arg1, arg2) */
#define LOG_INFO(logmsgFormat, ...) \
    do { \
        Logger& logger = Logger::instance(); \
        logger.setLogLevel(INFO); \
        char buf[1024] = {0}; \
        snprintf(buf, 1024, logmsgFormat, ##__VA_ARGS__); \
        logger.log(buf); \
    } while(0)
/*
    snprintf将格式化的字符串拷贝到buf中
    ##__VA_ARGS__表示可变参数 如果可变参数为空 它会自动去掉前面的逗号
*/

#define LOG_ERROR(logmsgFormat, ...) \
    do { \
        Logger& logger = Logger::instance(); \
        logger.setLogLevel(ERROR); \
        char buf[1024] = {0}; \
        snprintf(buf, 1024, logmsgFormat, ##__VA_ARGS__); \
        logger.log(buf); \
    } while(0)

#define LOG_FATAL(logmsgFormat, ...) \
    do { \
        Logger& logger = Logger::instance(); \
        logger.setLogLevel(FATAL); \
        char buf[1024] = {0}; \
        snprintf(buf, 1024, logmsgFormat, ##__VA_ARGS__); \
        logger.log(buf); \
    } while(0)

/* 
    对于调试信息 我们仅在调试时进行使用 正常使用时输出大量调试信息会降低效率
    当指定了MUDEBUG时输出调试信息 未指定时不输出调试信息(使用空宏)
*/
#ifdef MUDEBUG
#define LOG_DEBUG(logmsgFormat, ...) \
    do { \
        Logger& logger = Logger::instance(); \
        logger.setLogLevel(DEBUG); \
        char buf[1024] = {0}; \
        snprintf(buf, 1024, logmsgFormat, ##__VA_ARGS__); \
        logger.log(buf); \
    } while(0)
#else
#define LOG_DEBUG(logmsgFormat, ...)
#endif


/* 定义日志的级别 */
enum LogLevel {
    INFO,   // 普通信息
    ERROR,  // 错误信息
    FATAL,  // core信息
    DEBUG   // 调试信息
};


/* 日志类 Logger 单例模式 不可拷贝构造及赋值*/
class Logger : noncopyable {
public:
    /* 获取Logger唯一的实例对象 */
    static Logger& instance();
    /* 设置日志级别 */
    void setLogLevel(int level);
    /* 写日志接口 */
    void log(std::string msg);
private:
    Logger() {}
    int logLevel_;
};


#endif // __LOGGER_HH_
```


`Logger.cc`  
```cpp
#include "Logger.hh"
#include "Timestamp.hh"

#include <iostream>

/* 获取Logger唯一的实例对象 */
Logger& Logger::instance() {
    static Logger logger;
    return logger;
}

/* 设置日志级别 */
void Logger::setLogLevel(int level){
    logLevel_ = level;
}

/* 写日志接口 */
void Logger::log(std::string msg){
    /* 日志格式： [级别信息] time : msg */
    switch (logLevel_) {
        case INFO: 
            std::cout << "[INFO]";
            break;
        case ERROR: 
            std::cout << "[ERROR]";
            break;
        case FATAL: 
            std::cout << "[FATAL]";
            break;
        case DEBUG: 
            std::cout << "[DEBUG]";
            break;
    }
    /* 打印time和msg */
    std::cout << Timestamp::now().toString() << " : " << msg << std::endl;
}

```