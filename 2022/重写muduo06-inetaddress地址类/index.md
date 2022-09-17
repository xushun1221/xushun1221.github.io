# 【重写muduo】06 - InetAddress地址类


这里没什么好写的，就是把`sockaddr_in`以及相关的函数进行封装，我们仅支持ipv4。


## 源码

`InetAddress.hh`  
```cpp
#ifndef   __INETADDRESS_HH_
#define   __INETADDRESS_HH_

#include <arpa/inet.h>
#include <string>

/* socket地址类型封装 */
class InetAddress {
public:
    /* 使用ip+port构造socket地址 */
    explicit InetAddress(uint16_t port, std::string ip = "127.0.0.1");
    /* 使用sockaddr_in直接构造socket地址 */
    explicit InetAddress(const sockaddr_in& addr) : addr_(addr) {}
    /* 获取ip字符串 */
    std::string toIp() const;
    /* 获取ip+port字符串 */
    std::string toIpPort() const;
    /* 获取port */
    uint16_t toPort() const;
    /* 获得内部的sockaddr_in */
    const sockaddr_in* getSockAddr() const { return &addr_; }
private:
    /* socket地址类型 仅支持ipv4 */
    sockaddr_in addr_;
};


#endif // __INETADDRESS_HH_
```



`InetAddress.cc`  
```cpp
#include "InetAddress.hh"

#include <string.h>

/* 使用ip+port构造socket地址 点分十进制*/
InetAddress::InetAddress(uint16_t port, std::string ip) {
    /* 置零 string.h */
    bzero(&addr_, sizeof(addr_));
    addr_.sin_family = AF_INET;
    addr_.sin_port = htons(port);
    int dst;
    inet_pton(AF_INET, ip.c_str(), (void*)&dst);
    addr_.sin_addr.s_addr = dst;
}

/* 获取ip字符串 点分十进制*/
std::string InetAddress::toIp() const {
    char buf[64] = {0};
    inet_ntop(AF_INET, &addr_.sin_addr.s_addr, buf, sizeof(buf));
    return buf;
}

/* 获取ip+port字符串 xxx.xxx.xxx.xxx:xxxx*/
std::string InetAddress::toIpPort() const {
    char buf[64] = {0};
    inet_ntop(AF_INET, &addr_.sin_addr.s_addr, buf, sizeof(buf));
    uint16_t port = ntohs(addr_.sin_port);
    sprintf(buf + strlen(buf), ":%u", port);
    return buf;
}

/* 获取port */
uint16_t InetAddress::toPort() const {
    return ntohs(addr_.sin_port);
}

```
