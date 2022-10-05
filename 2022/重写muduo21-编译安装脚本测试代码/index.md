# 【重写muduo】21 - 编译安装脚本&测试代码


## 编译安装脚本

我们的项目使用cmake管理，需要编译为一个.so动态库，动态库安装到系统`/usr/lib`目录下，头文件拷贝到`/usr/include/mymuduo`目录下，使用时包含头文件即可。

通过脚本自动编译安装。

`autobuild.sh`  
```shell
#!/bin/bash

set -e

# 创建build目录
if [ ! -d `pwd`/build ]; then
    mkdir `pwd`/build
fi

# 清理
rm -rf `pwd`/build/*

# 编译生成
cd `pwd`/build &&
    cmake .. &&
    make

cd ..

# 把头文件拷贝到/usr/include/mymuduo
if [ ! -d /usr/include/mymuduo ]; then
    mkdir /usr/include/mymuduo
fi

for header in `ls *.hh`
do
    cp $header /usr/include/mymuduo
done

# so库拷贝到/usr/lib
cp `pwd`/lib/libmymuduo.so /usr/lib

# 在环境变量中添加了新的so库 刷新一下动态库缓存
ldconfig
```

## 测试代码

我们还是写一个简单的回显服务器来测试mymuduo库。

`test_server.cc`  
```cpp
#include <mymuduo/TcpServer.hh>
#include <mymuduo/Logger.hh>
#include <string>
#include <functional>

/* test Echo Server using mymuduo */

class EchoServer {
public:
    EchoServer(EventLoop* loop, const InetAddress& addr, const std::string& name)
        : server_(loop, addr, name)
        , loop_(loop)
        {
            using namespace std::placeholders;
            server_.setConnectionCallback(std::bind(&EchoServer::onConnection, this, _1));
            server_.setMessageCallback(std::bind(&EchoServer::onMessage, this, _1, _2, _3));
            server_.setThreadNum(3); /* 3 subloop */
    }
    void start() { server_.start(); }
private:
    /* 连接建立和关闭事件回调 */
    void onConnection(const TcpConnectionPtr& conn) {
        if (conn->connected()) {
            LOG_INFO("Connection UP : %s", conn->peerAddress().toIpPort().c_str());
        } else {
            LOG_INFO("Connection DOWN : %s", conn->peerAddress().toIpPort().c_str());
        }
    }
    /* 消息回调 */
    void onMessage(const TcpConnectionPtr& conn, Buffer* buffer, Timestamp time) {
        std::string msg = buffer->retrieveAllAsString();
        conn->send(msg); /* 回显 */
        conn->shutdown();
    }
    EventLoop* loop_;  /* mainloop */
    TcpServer server_;
};


int main() {

    EventLoop loop; /* mainloop */;
    InetAddress addr(8000);
    EchoServer server(&loop, addr, "EchoServer-01");
    server.start();
    loop.loop();    /* start mainloop epoll_wait */
    return 0;
}
```

在写一个makefile文件方便用户编译测试代码。

`makefile`  
```makefile
test_server :
	g++ -o test_server test_server.cc -lmymuduo -lpthread -g

clean :
	rm -f test_server
```

### 测试

编译：  
```shell
xushun@xushun-virtual-machine:~/mymuduo$ cd example/
xushun@xushun-virtual-machine:~/mymuduo/example$ make
g++ -o test_server test_server.cc -lmymuduo -lpthread
xushun@xushun-virtual-machine:~/mymuduo/example$ 
```

测试(用另一个终端telnet连接该服务器)：  
```shell
xushun@xushun-virtual-machine:~/mymuduo/example$ ./test_server 
[INFO]2022/10/05 08:55:38 1664931338.569048 : func=updateChannel => fd=4 events=3 index=-1 

[INFO]2022/10/05 08:55:38 1664931338.569779 : func=updateChannel => fd=7 events=3 index=-1 

[INFO]2022/10/05 08:55:38 1664931338.569854 : EventLoop 0x7f51da5e4c40 start looping 

[INFO]2022/10/05 08:55:38 1664931338.569892 : func=poll => fd total count:1

[INFO]2022/10/05 08:55:38 1664931338.570076 : func=updateChannel => fd=9 events=3 index=-1 

[INFO]2022/10/05 08:55:38 1664931338.570144 : EventLoop 0x7f51d9de3c40 start looping 

[INFO]2022/10/05 08:55:38 1664931338.570179 : func=poll => fd total count:1

[INFO]2022/10/05 08:55:38 1664931338.570263 : func=updateChannel => fd=11 events=3 index=-1 

[INFO]2022/10/05 08:55:38 1664931338.570329 : EventLoop 0x7f51d95e2c40 start looping 

[INFO]2022/10/05 08:55:38 1664931338.570362 : func=poll => fd total count:1

[INFO]2022/10/05 08:55:38 1664931338.570389 : func=updateChannel => fd=5 events=3 index=-1 

[INFO]2022/10/05 08:55:38 1664931338.570446 : EventLoop 0x7ffc90426170 start looping 

[INFO]2022/10/05 08:55:38 1664931338.570477 : func=poll => fd total count:2

[INFO]2022/10/05 08:55:42 1664931342.089791 : func=poll => 1 events happened

[INFO]2022/10/05 08:55:42 1664931342.089858 : Channel handleEvent revents: 1
[INFO]2022/10/05 08:55:42 1664931342.089947 : TcpServer::newConnection [EchoServer-01] - new connection [EchoServer-01-127.0.0.1:8000#1] from 127.0.0.1:51088 

[INFO]2022/10/05 08:55:42 1664931342.090077 : TcpConnection create[EchoServer-01-127.0.0.1:8000#1] at fd=12 

[INFO]2022/10/05 08:55:42 1664931342.090246 : func=poll => fd total count:2

[INFO]2022/10/05 08:55:42 1664931342.090273 : func=poll => 1 events happened

[INFO]2022/10/05 08:55:42 1664931342.090298 : Channel handleEvent revents: 1
[INFO]2022/10/05 08:55:42 1664931342.090351 : func=updateChannel => fd=12 events=3 index=-1 

[INFO]2022/10/05 08:55:42 1664931342.090381 : Connection UP : 127.0.0.1:51088
[INFO]2022/10/05 08:55:42 1664931342.090396 : func=poll => fd total count:2

[INFO]2022/10/05 08:55:44 1664931344.086311 : func=poll => 1 events happened

[INFO]2022/10/05 08:55:44 1664931344.086381 : Channel handleEvent revents: 1
[INFO]2022/10/05 08:55:44 1664931344.086564 : func=poll => fd total count:2

[INFO]2022/10/05 08:55:44 1664931344.086683 : func=poll => 1 events happened

[INFO]2022/10/05 08:55:44 1664931344.086704 : Channel handleEvent revents: 17
[INFO]2022/10/05 08:55:44 1664931344.086723 : fd=12 state=3 

[INFO]2022/10/05 08:55:44 1664931344.086737 : func=updateChannel => fd=12 events=0 index=1 

[INFO]2022/10/05 08:55:44 1664931344.086759 : Connection DOWN : 127.0.0.1:51088
[INFO][INFO]2022/10/05 08:55:44 1664931344.086871 : func=poll => fd total count:2

2022/10/05 08:55:44 1664931344.086877 : func=poll => 1 events happened

[INFO]2022/10/05 08:55:44 1664931344.086931 : Channel handleEvent revents: 1
[INFO]2022/10/05 08:55:44 1664931344.086957 : TcpServer::removeConnectionInLoop [EchoServer-01] - connection EchoServer-01-127.0.0.1:8000#1

[INFO]2022/10/05 08:55:44 1664931344.087026 : func=poll => fd total count:2

[INFO]2022/10/05 08:55:44 1664931344.087060 : func=poll => 1 events happened

[INFO]2022/10/05 08:55:44 1664931344.087097 : Channel handleEvent revents: 1
[INFO]2022/10/05 08:55:44 1664931344.087130 : func=removeChannel => fd=12 

[INFO]2022/10/05 08:55:44 1664931344.087166 : TcpConnection destroyed[EchoServer-01-127.0.0.1:8000#1] at fd=12 state=0 

[INFO]2022/10/05 08:55:44 1664931344.087223 : func=poll => fd total count:1

^C
```
可以成功回显，多个客户端清空也测试成功。
