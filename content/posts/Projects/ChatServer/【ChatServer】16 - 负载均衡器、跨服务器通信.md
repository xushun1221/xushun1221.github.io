---
title: 【ChatServer】16 - 负载均衡器、跨服务器通信
date: 2022-12-03
tags: [Linux, C++, Nginx, Redis]
categories: [Projects]
---

## 负载均衡器

对于单服务器而言，能够提供的并发服务量大概2万左右，如果需要更多的并发量，就需要多台服务器组成集群。

这就需要负载均衡器了，它将客户端请求通过负载均衡器发送到不同的服务器上。

## 跨服务器通信

如果两个客户端分别连接到不同的服务器上，如果这两个客户端需要进行通信，如果不引入中间件，那么势必要在服务器之间建立TCP通信。

这样的设计也会导致各个服务器之间耦合度太高，不利于系统扩展，占用大量的socket资源，各服务器之间的带宽压力很大，绝不是一个好的设计。

集群部署的服务器之间进行通信，最好的方式就是引入中间件消息队列，解耦多个服务器，使整个系统低耦合，提高服务响应能力，节省带宽资源。

在我们的这个项目中使用基于发布-订阅模式的Redis。


## Nginx TCP负载均衡配置

### 安装 Nginx

下载并解压`nginx-1.12.2.tar.gz`。

`configure --with-stream`，加入TCP负载均衡器模块。

`sudo make`

`sudo make install`

安装在`/usr/local/nginx`下，可执行文件在`sbin`目录中，配置文件在`conf/nginx.conf`中。

### 配置TCP负载均衡器

在配置文件中添加：  
```
# tcp
stream {
    upstream MyServer {
        server 127.0.0.1:6000 weight=1 max_fails=3 fail timeout=30s;
        server 127.0.0.1:6001 weight=1 max_fails=3 fail timeout=30s;
    }
    server {
        listen 8000;
        proxy_pass MyServer;
    }
}
```

`MyServer`就是工作服务器，添加服务器时在后面继续写就行。 这里已经添加了两个服务器，分别在6000和6001端口上。 
这里使用的是`8000`端口，服务器发送请求就使用这个端口。

`nginx -s reload`，平滑重启，重新加载配置文件并启动新添加的服务器。  
`nginx -s stop`，停止。

启动：  
```bash
[xushun@localhost chat]$ sudo /usr/local/nginx/sbin/nginx
[sudo] password for xushun: 
```

测试一下，开两个服务器，然后开几个客户端连接8000端口，可以看到两个服务器都有输出。



## Redis 基于发布-订阅的消息队列

### Redis 安装

下载解压`redis-6.2.7.tar.gz`

`make`编译

`sudo make PREFIX=/usr/local/redis install`安装

`/usr/local/redis/bin/redis-server`启动。


### 消息队列逻辑

1. 当某个客户端登录到某个服务器上时，该服务器应该以该客户的id作为通道号，在redis上订阅消息。
2. 如果服务器收到发送给某个客户的消息时，如果该客户连接不在该服务器上（在另一个服务器上），则以目标客户的id作为通道号，发送消息到redis上，另一个服务器就会收到这条客户消息。


### Redis客户端编程

C++对应的编程接口是`hiredis`，直接下载源码`https://github.com/redis/hiredis/archive/refs/tags/v1.1.0.tar.gz`。

`make`

`sudo make install`

动态库在`/usr/local/lib`下。

`sudo ldconfig /usr/local/lib`刷新缓存。


## 发布-订阅功能实现

对于和Redis进行交互的模块，我们也封装一个`Redis`类来实现。

```cpp
/*
 *  @Filename : Redis.hh
 *  @Description : Definition of Redis
 *  @Datatime : 2022/12/04 17:01:48
 *  @Author : xushun
 */
#ifndef __REDIS_HH_
#define __REDIS_HH_

#include <hiredis/hiredis.h>
#include <functional>
#include <thread>

class Redis
{
public:
    Redis();
    ~Redis();

    // 连接redis服务器
    bool connect();
    // 向指定channel发布消息
    bool publish(uint32_t channel, std::string message);
    // 向指定channel订阅消息
    bool subscribe(uint32_t channel);
    // 向指定channel取消订阅消息
    bool unsubscribe(uint32_t channel);
    // 在独立线程中接收该订阅通道中的消息
    void observerChannelMessage();
    // 初始化向业务层上报通道消息的回调对象
    void initNotifyHandler(std::function<void(uint32_t, std::string)> fn);

private:
    // hiredis同步上下文对象 相当于一个redisclient
    // 负责publish消息
    redisContext *publishContext_;
    // subscribe消息
    redisContext *subscribeContext_;
    // 回调方法 收到订阅的消息 给服务层上报
    std::function<void(uint32_t, std::string)> notifyMessageHandler_;
};

#endif // __REDIS_HH_
```