---
title: Linux下使用VSCode和CMake进行C++开发【1】
date: 2022-02-12
tags: [Linux, CMake, C++]
categories: [Coding]
---

## 安装Linux系统
这里用的是虚拟机
> Ubuntu 18.04 LTS

安装过程略。

-----

## 安装GCC、GDB、CMake

安装GCC，GDB
```shell
sudo apt update

sudo apt install build-essential gdb
```

安装CMake
```shell
sudo apt install cmake
```

确认安装成功
```shell
gcc --version
g++ --version
gdb --version
cmake --version
```

-----

## GCC编译器介绍
GCC编译器支持多种语言的编译，使用Linux开发C/C++需要熟悉GCC，VSCode也是通过调用GCC编译器来实现C/C++的编译工作的。
- 使用gcc命令编译C代码
- 使用g++命令编译C++代码

### 编译过程
1. 预处理 Pre-Processing	.i文件
```shell
# -E 选项表示编译器只对输入文件进行预处理
g++ -E test.cpp -o test.i
```

2. 编译 Compiling	.s文件
```shell
# -S 选项表示g++在为C++代码生成汇编文件后停止编译
# g++生成的汇编语言文件的缺省扩展名是.s
g++ -S test.i -o test.s
```

3. 汇编 Assembling	.o文件
```shell
# -c 选项表示g++仅把源代码编译为机器语言的目标代码
# 缺省时g++建立的目标代码文件扩展名是.o
g++ -c test.s -o test.o
```

4. 链接 Linking	bin文件
```shell
# -o 选项表示为将生成的可执行文件使用指定的文件名
g++ test.o -o test
```

也可以使用简略写法，自动完成上述步骤
```shell
g++ test.cpp -o test
```

-----

### g++重要的编译参数
#### `-g`	编译带有调试嘻嘻的可执行文件
```shell
# -g 表示g++生成可以被GNU调试器GDB使用的调试信息，以便调试程序
# 生成带调试信息的可执行文件test
g++ -g test.cpp -o test
```

####  `-O[n]`	优化源代码
优化指进行一些操作，以缩减目标文件所包含的代码量，提高最终生成的可执行文件的运行效率，如省略代码中从未使用的变量、将常量表达式用结果值代替等。
```shell
# -O 选项表示g++对源代码进行基本优化 -O2 选项表示g++生成尽可能小和尽可能块的代码，通常使用O2即可。
# -O 同时减小代码长度和执行时间，等价于O1
# -O0 不做优化
# -O1 默认优化
# -O2 除了完成 -O1 的优化之外，还进行一些额外的调整工作，如指令调整等
# -O3 包括循环展开和其他一些与处理特性相关的优化工作

# 使用 -O2 优化源代码，并输出可执行文件
g++ -O2 test.cpp -o test
```
*可以通过time指令查看执行时间，`time ./test`*

#### `-l` 和 `-L`	指定库文件 | 指定库文件路径
`-l` 选项是用来指定程序要链接的库，`-l`后紧跟库名，在`/lib`和`/usr/lib`以及`usr/local/lib`中的库就可以直接使用`-l`选项链接。
```shell
# 链接glog库
g++ -lglog test.cpp
```
如果库文件没有在上述三个目录中，需要使用`-L`选项指定库文件所在目录，`-L`后紧跟库文件目录。
```shell
# 链接mytest库，libmytest.so在/home/xushun/mytetlibfolder目录下
g++ -L/home/xushun/mytestlibfolder -lmytest test.cpp
```

#### `-I`	指定头文件搜索目录
 `/usr/include`目录下的头文件一般不用指定的，如果头文件不在该目录下，我们就需要用`I`选项指定头文件的目录，如头文件放在`/myinclude`目录下，那编译指令加上`-I/myinclude`即可，也可以使用相对路径，如头文件在当前目录下，则只需`-I.`即可。
 ```shell
 g++ -I/myinclude test.cpp
 ```
 
 #### `-Wall`	打印警告信息
 ```shell
 # 打印g++提供的警告信息
 g++ -Wall test.cpp
 ```
 
 #### `-w`	关闭警告信息
 ```shell
 # 关闭所有警告信息
 g++ -w test.cpp
 ```
 
 #### `-std=c++11`	设置编译标准
 ```shell
 # 使用c++11标准编译test.cpp
 g++ -std=c++11 test.cpp
 ```
 
 #### `-o`	指定输出文件名
 ```shell
 # 指定即将生成的文件名
 g++ test.cpp -o test
 ```
 
 #### `-D`	定义宏
 在使用gcc/g++编译的使用定义宏，常用的场景：`-DDEBUG`定义DEBUG宏，可能文件中有DEBUG宏部分的相关信息，用DDEBUG来选择开启或关闭DEBUG。
 示例代码：
 ```cpp
#include <stdio.h>
int main() {
	#ifdef DEBUG
		printf("DEBUG LOG\n");
	#endif
	printf("in\n");
}

// 如果在编译的时候使用 g++ -DDEBUG main.cpp
// 语句printf("DEBUG LOG\n");则可以被执行
```

#### gcc使用手册
可以使用`man gcc`指令查看gcc的英文使用手册。

-----

## GCC编译示例
我们来看一下怎么编译一个简单的项目。

先写一个简单的项目，目录结构如下：
![](/post_images/posts/Coding/Linux下使用VSCode和CMake进行C++开发【1】/目录.jpg "目录结构")
![](/post_images/posts/Coding/Linux下使用VSCode和CMake进行C++开发【1】/main.jpg "main.cpp")
![](/post_images/posts/Coding/Linux下使用VSCode和CMake进行C++开发【1】/swaph.jpg "swap.h")
![](/post_images/posts/Coding/Linux下使用VSCode和CMake进行C++开发【1】/swapcpp.jpg "swap.cpp")

### 直接编译
直接编译会报错：
```shell
g++ main.cpp src/swap.cpp -o demo
```

需要加上头文件目录：
```shell
g++ main.cpp src/swap.cpp -Iinclude -o demo
```

### 生成静态库文件 .a
进入src目录下
```shell
# 汇编，生成swap.o文件
g++ swap.cpp -c -I../include
# 生成静态库文件libswap.a
ar rs libswap.a swap.o
```

生成了静态库文件，我们就可以把`main.cpp`和静态库链接起来，生成可执行文件`static_main`
```shell
# 切换到demo目录下
g++ main.cpp -lswap -Lsrc -Iinclude -o static_main
```

这时，我们运行可执行文件，可以正确运行，因为静态库在编译时已经链接了。

### 生成动态库文件 .so
进入src目录下
```shell
# 生成swap.o
g++ swap.cpp -c -I../include -fPIC
# 生成动态库文件libswap.so
g++ -shared -o libswap.so swap.o


# 也可以将上面两条指令合并
g++ swap.cpp -I../include -fPIC -shared -o libswap.so
```

生成了动态库文件，我们可以把`main.cpp`和动态库链接起来，生成可执行文件`dyna_main`
```shell
# 切换到demo目录下
g++ main.cpp -Iinclude -Lsrc -lswap -o dyna_main
```

现在，我们运行这个可执行文件会报错，可执行文件找不到动态库。

### 运行可执行文件
- 静态库：直接运行
```shell
./static_main
```

- 动态库：需要给定动态库的目录
```shell
LD_LIBRARY_PATH=src ./dyna_main
```
这样链接到动态库的可执行文件也可以正确运行了。

-----

## 参考资料
1. https://www.bilibili.com/video/BV1fy4y1b7TC