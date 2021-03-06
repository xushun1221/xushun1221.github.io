---
title: 【Linux系统编程】03 - gcc编译器 静态库和动态库
date: 2022-05-12
tags: [Linux, gcc]
categories: [Coding]
---

gcc编译器可以参考[gcc编译器](https://xushun1221.github.io/2022/linux%E4%B8%8B%E4%BD%BF%E7%94%A8vscode%E5%92%8Ccmake%E8%BF%9B%E8%A1%8Cc-%E5%BC%80%E5%8F%911/)  
gdb调试器可以参考[gdb调试器](https://xushun1221.github.io/2022/linux%E4%B8%8B%E4%BD%BF%E7%94%A8vscode%E5%92%8Ccmake%E8%BF%9B%E8%A1%8Cc-%E5%BC%80%E5%8F%912/)

## gcc编译的四步骤
源文件（`hello.c`）编译为可执行文件（`a.out`）步骤：
1. 预处理(`gcc hello.c -E`)：展开宏和头文件，替换条件编译，删除注释和空白；
2. 编译(`gcc hello.i -S`)：检查语法规范（消耗时间和系统资源最多）；
3. 汇编(`gcc hello.s -c`)：汇编指令翻译为机器指令；
4. 链接(`gcc hello.o 无参数`)：数据段合并，地址回填。

## gcc编译的重要参数
主要掌握`-I -c -g -Wall -D`这几个。  
这里做过笔记，就不在赘述了，[gcc编译重要参数](https://xushun1221.github.io/2022/linux%E4%B8%8B%E4%BD%BF%E7%94%A8vscode%E5%92%8Ccmake%E8%BF%9B%E8%A1%8Cc-%E5%BC%80%E5%8F%911/#g%E9%87%8D%E8%A6%81%E7%9A%84%E7%BC%96%E8%AF%91%E5%8F%82%E6%95%B0)。

## 静态库和共享库（动态库）
- 静态库文件需要和应用程序一起编译到可执行文件中；
- 共享库无需和应用程序代码绑定，多个应用程序可以共享一个库文件，当程序某个函数调用动态库时，再从动态库中将代码加载即可；
- 静态库用在对空间要求较低，而对时间要求较高的核心程序中；
- 动态库用在对时间要求较低，而对空间要求较高的程序中。

动态库的使用相对较多。

## 静态库的制作和使用
静态库名字通常以`lib`开头，以`.a`结尾，例如`libmymath.a`。  
步骤：
1. 编写好源代码，假设这里编写好了三个文件`add.c sub.c  div1.c`，三个文件中各自有一个算数函数（`int add(int a, int b) {...} ...`）；
2. 将源代码编译为`.o`文件，使用命令`gcc -c add.c -o add.o`，其他两个一样；
3. 使用`ar`工具制作静态库，`ar rs libmymath.a add.o sub.o div1.o`，将上面三个目标文件，放到名为`mymath`的静态库中（调用这个库时，将其称呼为`mymath`而不是`libmymath.a`）;
4. 使用静态库，用一个测试程序`test.c`调用这三个算术函数，编译时将静态库和源代码一起编译，`gcc test.c libmymath.a -o test`（源码要写在前面）。

可以结合这个一起看[静态库文件制作](https://xushun1221.github.io/2022/linux%E4%B8%8B%E4%BD%BF%E7%94%A8vscode%E5%92%8Ccmake%E8%BF%9B%E8%A1%8Cc-%E5%BC%80%E5%8F%911/#%E7%94%9F%E6%88%90%E9%9D%99%E6%80%81%E5%BA%93%E6%96%87%E4%BB%B6-a)

## 静态库使用及头文件对应
如果编译上面的代码，编译器会报警告，隐式声明警告。  
编译过程中，在函数调用之前，如果编译器没有见到过该函数的声明或定义，编译器就会帮你进行隐式的声明（编译器只会进行返回值为int的隐式声明，这里正好和我们的算术函数相同，所以进行了隐式声明，如果我们的函数长这样`void *add(int, int);`，就会报错误）。

在`test.c`源文件中，加入函数的声明即可解决这个问题。（不好的方式）

合理的方式应该是使用将`mymath`静态库中的函数声明写入头文件中`mymath.h`，在调用时包含该头文件即可`#include "mymath.h"`。  
头文件写法要注意：  
```c
#ifndef _MYMATH_H_
#define _MYMATH_H_

int add(int, int);
int sub(int, int);
int div1(int, int);

#endif
```
这样的写法好处在于，无论包含该头文件多少次，该头文件只会被展开一次。第二次进入该头文件时就会判断`_MYMATH_H_`已经定义了，直接跳转到`#endif`。

## 动态库制作和使用
制作动态库时，将`.c`文件编译为`.o`文件时，必须加上`-fPIC`选项，这和地址回填有关。  
步骤：
1. 编写源代码；
2. 将源代码编译为`.o`文件，`gcc -c add.c -fPIC -o add.o`；
3. 使用`-shared`选项制作动态库，`gcc -shared libmymath.so add.o sub.o div1.o`，动态库一般以`lib`开头，`.so`结尾；
4. 编译可执行程序时，指定所使用的动态库，`-l`指定库名，`-L`指定库路径，`gcc test.c -o test -lmymath -L./lib`，假设库在`./lib`目录下；
5. `test`可执行文件成功生成，但是无法执行。

执行出错的原因：
- 链接器：工作于链接阶段，工作时需要`-l`和`-L`选项支持；
- 动态链接器：工作于程序运行阶段，工作时需要提供动态库所在目录位置。

解决方法：
1. 通过环境变量`LD_LIBRARY_PATH`指定动态库位置，`export LD_LIBRARY_PATH=xxx`（要使用绝对路径），然后`./test`即可执行；(环境变量是进程的概念，切换终端后就会失效，如果希望永久生效，需要改动`~/.bashrc`配置文件，并运行`. .bashrc`，或者直接重启终端，每次启动终端时都会自动加载该配置文件)
2. 这样好像也行，`LD_LIBRARY_PATH=xxx ./test`；
3. 热知识，我们使用的标准c库也是动态库，使用时不需要设置该环境变量，因为它在`/lib`目录下的某个地方，我们可以把我们的动态库放到`/lib`目录下，运行时就无需设置环境变量，动态链接器会自动搜索`/lib`下的动态库；
4. `sudo vi /etc/ld.so.conf`打开这个文件，写入动态库的绝对路径，保存，`sudo ldconfig -v`使配置文件生效。
5. （冷知识，`ldd test`命令可以查看程序运行时需要的动态库以及它们的位置）