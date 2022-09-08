---
title: 【重写muduo】01 - 安装和使用 muduo
date: 2022-09-07
tags: [network, C++]
categories: [Networking]
---

## 安装 muduo

先把muduo和boost的源码下载下来（muduo需要依赖boost）。

### boost 源码库编译安装

先解压：  
```shell
xushun@xushun-virtual-machine:~/Downloads$ tar -zxvf boost_1_69_0.tar.gz
...
xushun@xushun-virtual-machine:~/Downloads$ cd boost_1_69_0/
xushun@xushun-virtual-machine:~/Downloads/boost_1_69_0$ ls
boost            boost.css      bootstrap.sh  index.html  libs             rst.css
boost-build.jam  boost.png      doc           INSTALL     LICENSE_1_0.txt  status
boostcpp.jam     bootstrap.bat  index.htm     Jamroot     more             tools
```

运行`bootstrap.sh`工程编译构建程序：  
```shell
xushun@xushun-virtual-machine:~/Downloads/boost_1_69_0$ ./bootstrap.sh 
Building Boost.Build engine with toolset gcc... tools/build/src/engine/bin.linuxx86_64/b2
Unicode/ICU support for Boost.Regex?... not found.
Generating Boost.Build configuration in project-config.jam...

Bootstrapping is done. To build, run:

    ./b2
    
To adjust configuration, edit 'project-config.jam'.
Further information:

   - Command line help:
     ./b2 --help
     
   - Getting started guide: 
     http://www.boost.org/more/getting_started/unix-variants.html
     
   - Boost.Build documentation:
     http://www.boost.org/build/doc/html/index.html

xushun@xushun-virtual-machine:~/Downloads/boost_1_69_0$ ls
b2               boostcpp.jam   bootstrap.log  index.html  LICENSE_1_0.txt     status
bjam             boost.css      bootstrap.sh   INSTALL     more                tools
boost            boost.png      doc            Jamroot     project-config.jam
boost-build.jam  bootstrap.bat  index.htm      libs        rst.css
```

生成了`b2`构建程序，运行之：  
```shell
xushun@xushun-virtual-machine:~/Downloads/boost_1_69_0$ ./b2
```

编译成功后会显示：  
```shell
The Boost C++ Libraries were successfully built!

The following directory should be added to compiler include paths:

    /home/xushun/Downloads/boost_1_69_0

The following directory should be added to linker library paths:

    /home/xushun/Downloads/boost_1_69_0/stage/lib
```

然后需要将`boost`库头文件和lib库文件安装在默认linux系统头文件和库文件搜索路径下，运行下面的命令：  
```shell
xushun@xushun-virtual-machine:~/Downloads/boost_1_69_0$ sudo ./b2 install
```

完成后会显示：  
```shell
...
common.copy /usr/local/lib/libboost_wave.so.1.69.0
ln-UNIX /usr/local/lib/libboost_wave.so
common.copy /usr/local/lib/libboost_exception.a
common.copy /usr/local/lib/libboost_system.a
common.copy /usr/local/lib/libboost_chrono.a
common.copy /usr/local/lib/libboost_timer.a
common.copy /usr/local/lib/libboost_test_exec_monitor.a
...updated 14831 targets...

```

安装完成，编译下面的代码测试一下：  
```C++
#include <iostream>
#include <boost/bind.hpp>
#include <string>
using namespace std;
class Hello {
public:
	void say(string name) { cout << name << " say: hello world!" << endl; }
};
int main() {
	Hello h;
	auto func = boost::bind(&Hello::say, &h, "zhang san");
	func();
	return 0;
}
```

输出：  
```shell
xushun@xushun-virtual-machine:~/Downloads$ touch testboost.cpp
xushun@xushun-virtual-machine:~/Downloads$ gedit testboost.cpp 
xushun@xushun-virtual-machine:~/Downloads$ g++ testboost.cpp -o testboost
xushun@xushun-virtual-machine:~/Downloads$ ./testboost 
zhang san say: hello world!
```

安装完成。


### 编译 muduo 网络库

先解压：  
```shell
xushun@xushun-virtual-machine:~/Downloads$ unzip muduo-master.zip 
xushun@xushun-virtual-machine:~/Downloads$ cd muduo-master/
xushun@xushun-virtual-machine:~/Downloads/muduo-master$ ls
BUILD.bazel  ChangeLog   CMakeLists.txt  examples  muduo    README
build.sh     ChangeLog2  contrib         License   patches  WORKSPACE
```

`muduo`库源码编译时会编译很多测试用例代码，耗时较长，我们用不到，`CMakeLists.txt`文件修改如下：  
```cmake
cmake_minimum_required(VERSION 2.6)

project(muduo C CXX)

enable_testing()

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release")
endif()

# only build examples if this is the main project
if(CMAKE_PROJECT_NAME STREQUAL "muduo")
#  option(MUDUO_BUILD_EXAMPLES "Build Muduo examples" ON) 注释这一行 不生成测试用例
endif()
```

`muduo`使用`cmake`进行构建，需要先安装`cmake`，我已经安装过了。

执行`build.sh`进行编译：  
```shell
xushun@xushun-virtual-machine:~/Downloads/muduo-master$ ./build.sh
...
[ 91%] Building CXX object muduo/net/inspect/CMakeFiles/muduo_inspect.dir/Inspector.cc.o
[ 93%] Building CXX object muduo/net/inspect/CMakeFiles/muduo_inspect.dir/PerformanceInspector.cc.o
[ 95%] Building CXX object muduo/net/inspect/CMakeFiles/muduo_inspect.dir/ProcessInspector.cc.o
[ 97%] Building CXX object muduo/net/inspect/CMakeFiles/muduo_inspect.dir/SystemInspector.cc.o
[100%] Linking CXX static library ../../../lib/libmuduo_inspect.a
[100%] Built target muduo_inspect
```

编译后安装：  
```shell
xushun@xushun-virtual-machine:~/Downloads/muduo-master$ ./build.sh install
...
```
这个`./build.sh install`实际上把`muduo`的头文件和lib库文件放到了`muduo-master`同级目录下的`build`目录下的`release-install-cpp11`文件夹下面了，我们把它们拷贝到系统默认搜索路径下。

```shell
xushun@xushun-virtual-machine:~/Downloads/muduo-master$ cd ../build/
xushun@xushun-virtual-machine:~/Downloads/build$ ls
release-cpp11  release-install-cpp11
xushun@xushun-virtual-machine:~/Downloads/build$ cd release-install-cpp11/
xushun@xushun-virtual-machine:~/Downloads/build/release-install-cpp11$ ls
include  lib
xushun@xushun-virtual-machine:~/Downloads/build/release-install-cpp11$ cd include/
xushun@xushun-virtual-machine:~/Downloads/build/release-install-cpp11/include$ ls
muduo
xushun@xushun-virtual-machine:~/Downloads/build/release-install-cpp11/include$ sudo mv muduo/ /usr/include/
[sudo] password for xushun: 
xushun@xushun-virtual-machine:~/Downloads/build/release-install-cpp11/include$ cd ..
xushun@xushun-virtual-machine:~/Downloads/build/release-install-cpp11$ ls
include  lib
xushun@xushun-virtual-machine:~/Downloads/build/release-install-cpp11$ cd lib/
xushun@xushun-virtual-machine:~/Downloads/build/release-install-cpp11/lib$ ls
libmuduo_base.a  libmuduo_http.a  libmuduo_inspect.a  libmuduo_net.a
xushun@xushun-virtual-machine:~/Downloads/build/release-install-cpp11/lib$ sudo mv * /usr/local/lib
```

安装完成，测试：  
```C++
#include <muduo/net/TcpServer.h>
#include <muduo/base/Logging.h>
#include <boost/bind.hpp>
#include <muduo/net/EventLoop.h>

// 使用muduo开发回显服务器
class EchoServer
{
 public:
  EchoServer(muduo::net::EventLoop* loop,
             const muduo::net::InetAddress& listenAddr);

  void start(); 

 private:
  void onConnection(const muduo::net::TcpConnectionPtr& conn);

  void onMessage(const muduo::net::TcpConnectionPtr& conn,
                 muduo::net::Buffer* buf,
                 muduo::Timestamp time);

  muduo::net::TcpServer server_;
};

EchoServer::EchoServer(muduo::net::EventLoop* loop,
                       const muduo::net::InetAddress& listenAddr)
  : server_(loop, listenAddr, "EchoServer")
{
  server_.setConnectionCallback(
      boost::bind(&EchoServer::onConnection, this, _1));
  server_.setMessageCallback(
      boost::bind(&EchoServer::onMessage, this, _1, _2, _3));
}

void EchoServer::start()
{
  server_.start();
}

void EchoServer::onConnection(const muduo::net::TcpConnectionPtr& conn)
{
  LOG_INFO << "EchoServer - " << conn->peerAddress().toIpPort() << " -> "
           << conn->localAddress().toIpPort() << " is "
           << (conn->connected() ? "UP" : "DOWN");
}

void EchoServer::onMessage(const muduo::net::TcpConnectionPtr& conn,
                           muduo::net::Buffer* buf,
                           muduo::Timestamp time)
{
  // 接收到所有的消息，然后回显
  muduo::string msg(buf->retrieveAllAsString());
  LOG_INFO << conn->name() << " echo " << msg.size() << " bytes, "
           << "data received at " << time.toString();
  conn->send(msg);
}


int main()
{
  LOG_INFO << "pid = " << getpid();
  muduo::net::EventLoop loop;
  muduo::net::InetAddress listenAddr(8888);
  EchoServer server(&loop, listenAddr);
  server.start();
  loop.loop();
}

```

编译运行：  
```C++
xushun@xushun-virtual-machine:~/Downloads$ touch testmuduo.cpp
xushun@xushun-virtual-machine:~/Downloads$ gedit testmuduo.cpp 
xushun@xushun-virtual-machine:~/Downloads$ g++ testmuduo.cpp -lmuduo_net -lmuduo_base -lpthread -std=c++11
xushun@xushun-virtual-machine:~/Downloads$ ./a.out 
20220907 12:59:32.781524Z 42004 INFO  pid = 42004 - testmuduo.cpp:61
20220907 12:59:35.562126Z 42004 INFO  TcpServer::newConnection [EchoServer] - new connection [EchoServer-0.0.0.0:8888#1] from 127.0.0.1:36014 - TcpServer.cc:80
20220907 12:59:35.562184Z 42004 INFO  EchoServer - 127.0.0.1:36014 -> 127.0.0.1:8888 is UP - testmuduo.cpp:42
20220907 12:59:35.562244Z 42004 INFO  EchoServer-0.0.0.0:8888#1 echo 12 bytes, data received at 1662555575.562196 - testmuduo.cpp:53
20220907 12:59:41.336223Z 42004 INFO  EchoServer - 127.0.0.1:36014 -> 127.0.0.1:8888 is DOWN - testmuduo.cpp:42
20220907 12:59:41.336275Z 42004 INFO  TcpServer::removeConnectionInLoop [EchoServer] - connection EchoServer-0.0.0.0:8888#1 - TcpServer.cc:109
^C
```

开另一个shell：  
```shell
xushun@xushun-virtual-machine:~$ echo "hello world" | nc localhost 8888
hello world
^C
```

正确回显，测试成功，安装完成。



## muduo 服务器编程示例

muduo网络库主要给用户提供了两个主要的类：  
- `TcpServer`：用于编写服务器程序；
- `TcpClient`：用于编写客户端程序。

网络库主要的功能就是将`epoll`和线程池进行了封装，把网络IO的代码和业务逻辑代码进行区分。仅仅将用户的连接和断开以及用户的可读写事件暴露出来。















