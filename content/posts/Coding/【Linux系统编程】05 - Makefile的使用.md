---
title: 【Linux系统编程】05 - Makefile的使用
date: 2022-05-13
tags: [Linux, Makefile]
categories: [Coding]
---

## Makefile基础
Makefile是用来管理项目的脚本。

Makefile的命名只有两种方式：`makefile`，`Makefile`。

Makefile要掌握的几个要点：
1. 1个规则：生成目标的规则这样写，`目标:依赖条件(空格分开)\n\tab命令`。简单来说：
   1. 如果依赖条件不存在，就去寻找新的规则来产生依赖条件；
   2. 目标的修改时间必须晚于依赖条件的修改时间，否则，更新目标。
2. 2个函数：
3. 3个自动变量：

## Makefile 1个规则
1. 如果想生成目标，检查规则中的依赖条件是否存在，如不存在，则寻找是否有新规则用来生成该依赖文件；
2. 检查规则中的目标是否需要更新，必须检查它的所有依赖，依赖中有任一个被更新，则目标必须更新：
   1. 分析各个目标和依赖之间的关系；
   2. 根据依赖关系自底向上执行命令；
   3. 依赖修改时间比目标新，则确定更新；
   4. 如果目标不依赖任何条件，则执行相应命令，以示更新。

注意：make认为第一个目标就是最终目标，生成了第一个目标后，不会执行其他的命令。可以使用`ALL:xxx.out`命令来指定生成的最终目标，告诉make命令，无论第一条命令是什么，我要生成的最终目标都是`xxx.out`，这样，书写顺序就无所谓了。

### 测试项目
写一个简单的项目进行测试，项目结构：  
```shell
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ ls
add.c  div1.c  main.c  Makefile  sub.c
```
`add.c sub.c div1.c`分别包含计算加法，减法和除法的一个函数，`main`函数如下：  
```c
#include<stdio.h>
int add(int, int);
int sub(int, int);
int div1(int, int);
int main() {
    int a = 10;
    int b = 5;
    printf("a = %d, b = %d\n", a, b);
    printf("a + b = %d\n", add(a, b));
    printf("a - b = %d\n", sub(a, b));
    printf("a / b = %d\n", div1(a, b));
    return 0;
}
```

### 版本1：全部编译
`Makefile`文件内容如下：  
```makefile
ALL:test.out

test.out:main.c add.c sub.c div1.c
	gcc main.c add.c sub.c div1.c -o test.out
```
执行`make`命令，可以成功生成。

### 版本2：部分编译、链接
版本1可以实现生成功能，但是每次生成都需要编译全部的源码文件，消耗较大，我们可以用`.o`文件进行链接，只更新那些修改了`.c`依赖的`.o`文件，这样只会重新编译那些修改了的源码文件，可以提高编译速度。`Makefile`文件内容如下：  
```console
ALL:test.out

test.out:main.o add.o sub.o div1.o
	gcc main.o add.o sub.o div1.o -o test.out

main.o:main.c
	gcc -c main.c -o main.o
add.o:add.c
	gcc -c add.c -o add.o
sub.o:sub.c
	gcc -c sub.c -o sub.o
div1.o:div1.c
	gcc -c div1.c -o div1.o
```
执行`make`命令，成功生成目标文件：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ make
gcc -c main.c -o main.o
gcc -c add.c -o add.o
gcc -c sub.c -o sub.o
gcc -c div1.c -o div1.o
gcc main.o add.o sub.o div1.o -o test.out
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ ./test.out 
a = 10, b = 5
a + b = 15
a - b = 5
a / b = 2
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ ls
add.c  add.o  div1.c  div1.o  main.c  main.o  Makefile  sub.c  sub.o  test.out
```
我们修改`add`函数，让它在加上1，重新`make`，测试：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ make
gcc -c add.c -o add.o
gcc main.o add.o sub.o div1.o -o test.out
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ ls
add.c  add.o  div1.c  div1.o  main.c  main.o  Makefile  sub.c  sub.o  test.out
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ ./test.out 
a = 10, b = 5
a + b = 16
a - b = 5
a / b = 2
```
可以看到，`make`程序只程序编译了`add.c`文件，并进行了链接，其他未修改文件并没有重新编译。