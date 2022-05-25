# 【Linux系统编程】05 - Makefile的使用


## Makefile基础
Makefile是用来管理项目的脚本。

Makefile的命名只有两种方式：`makefile`，`Makefile`。

Makefile要掌握的几个要点：
1. 1个规则：生成目标的规则这样写，`目标:依赖条件(空格分开)\n\tab命令`。简单来说：
   1. 如果依赖条件不存在，就去寻找新的规则来产生依赖条件；
   2. 目标的修改时间必须晚于依赖条件的修改时间，否则，更新目标。
2. 2个函数：
   1. `src = $(wildcard *.c)`
   2. `obj = $(patsubst %.c, %.o, $(src))`
3. 3个自动变量：
   1. `$@`
   2. `$<`
   3. `$^`

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

## Makefile 2个函数和clean
两个函数：
1. `wildcard`，通配符函数，`src = $(wildcard ./*.c)`，（`src`是一个Makefile文件的变量，`$()`表示函数调用，`wildcard`表示函数名，`*.c`是该函数的参数），这句的意思是，找到当前目录下所有后缀为`.c`的文件，将文件名提取出来，组成一个列表，赋值给`src`；（在我们的测试项目目录下，相当于，`src = main.c add.c sub.c div1.c`）
2. `patsubst`，替换函数，`obj = $(patsubst %.c, %.o, $(src))`，（`$(变量)`是对变量取值，`%`是匹配符），这句的意思是，将参数3`$(src)`中，包含参数1`%.c`的部分，替换为参数2`%.o`。（相当于，`obj=main.o add.o sub.o div1.o`）

`clean`目标：`clean:(没有依赖条件)\n\tab-rm -rf $(obj) test.out`，这句意思是将所有的`.o`文件和最终目标`test.out`删除。

可以执行`make clean`来自动清除这些文件。

### 版本3：自动获取源文件列表以及清理中间文件
利用上面讲的两个函数获取源文件列表，以及`clean`方法清理中间文件。  
```makefile
src = $(wildcard *.c)
obj = $(patsubst %.c, %.o, $(src))

ALL:test.out

test.out:$(obj)
	gcc $(obj) -o test.out

main.o:main.c
	gcc -c main.c -o main.o
add.o:add.c
	gcc -c add.c -o add.o
sub.o:sub.c
	gcc -c sub.c -o sub.o
div1.o:div1.c
	gcc -c div1.c -o div1.o

clean:
	-rm -rf $(obj) test.out # rm之前的- 表示出错依然执行 当删除不存在的文件时 不会报错
```
删除之前生成的`.o`和out文件，测试：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ ls
add.c  div1.c  main.c  Makefile  sub.c
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ make
gcc -c div1.c -o div1.o
gcc -c main.c -o main.o
gcc -c add.c -o add.o
gcc -c sub.c -o sub.o
gcc  div1.o  main.o  add.o  sub.o -o test.out
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ ls
add.c  add.o  div1.c  div1.o  main.c  main.o  Makefile  sub.c  sub.o  test.out
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ ./test.out
a = 10, b = 5
a + b = 15
a - b = 5
a / b = 2
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ make clean -n
rm -rf  div1.o  add.o  main.o  sub.o test.out
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ make clean
rm -rf  div1.o  add.o  main.o  sub.o test.out
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ ls
add.c  div1.c  main.c  Makefile  sub.c
```
可以看到编译成功生成可执行文件，`make clean -n`命令是查看要删除的文件，防止错误删除操作，在`make clean`执行之前需要执行一次。

## Makefile 3个自动变量和模式规则
3个自动变量：
- `$@`，在规则的命令中（目标和依赖条件中不能出现），表示规则中的目标；
- `$^`，在规则的命令中，表示所有依赖条件；
- `$<`，在规则的命令中，表示第一个依赖条件。如果该变量应用在模式规则中，它可将依赖条件列表中的依赖依次取出，套用模式规则。

模式规则：
- 可以发现版本4的Makefile文件依然不是很完善，如果我们添加一个`mul.c`的乘法模块，源码可以被`src`自动识别，而编译为`.o`文件的过程不能自动进行；
- 可以用`%.o:%.c\n\tabgcc -c $< -o $@`来表示将所有`.c`文件编译为`.o`文件，在添加新的`.c`文件时，无需手动添加编译命令。

静态模式规则，指定该模式规则给谁用，这里指定这个模式规则给`$(obj)`用，只需要在开头加上`$(obj):`即可，`$(obj):%.o:%.c\n\tabgcc -c $< -o $@`。

### 版本4：引入自动变量
```makefile
src = $(wildcard *.c)
obj = $(patsubst %.c, %.o, $(src))

ALL:test.out

test.out:$(obj)
	gcc $^ -o $@

main.o:main.c
	gcc -c $< -o $@
add.o:add.c
	gcc -c $< -o $@
sub.o:sub.c
	gcc -c $< -o $@
div1.o:div1.c
	gcc -c $< -o $@

clean:
	-rm -rf $(obj) test.out
```
同样可以成功生成。

### 版本5：引入模式规则
```makefile
src = $(wildcard *.c)
obj = $(patsubst %.c, %.o, $(src))

ALL:test.out

test.out:$(obj)
	gcc $^ -o $@

%.o:%.c
	gcc -c $< -o $@

clean:
	-rm -rf $(obj) test.out
```
同样可以成功生成。这样之后添加其他`.c`模块进来就无需改动`Makefile`文件，直接`make`即可。

## 其他问题
### clean和all文件问题
如果当前目录下有`clean`或`ALL`文件，会影响`make`的指令判断，会觉得需要生成`clean`和`ALL`目标。  
解决这个问题需要将`clean`生成一个**伪目标**，不管条件是什么伪目标都要被执行。在`Makefile`文件最后加入一行`.PHONY: clean ALL`即可。

### 版本6：添加伪目标
```makefile
src = $(wildcard *.c)
obj = $(patsubst %.c, %.o, $(src))

ALL:test.out

test.out:$(obj)
	gcc $^ -o $@

%.o:%.c
	gcc -c $< -o $@

clean:
	-rm -rf $(obj) test.out

.PHONY: clean ALL
```
测试，可以看到即使存在`clean`文件，`make clean`命令也不会报错：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ ls
add.c  div1.c  main.c  Makefile  sub.c
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ touch clean
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ ls
add.c  clean  div1.c  main.c  Makefile  sub.c
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ make
gcc -c div1.c -o div1.o
gcc -c main.c -o main.o
gcc -c add.c -o add.o
gcc -c sub.c -o sub.o
gcc div1.o main.o add.o sub.o -o test.out
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ ./test.out
a = 10, b = 5
a + b = 15
a - b = 5
a / b = 2
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ make clean
rm -rf  div1.o  add.o  main.o  sub.o test.out
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ ls
add.c  clean  div1.c  main.c  Makefile  sub.c
```

### 添加编译参数
编译时可能会使用其他的编译参数，例如`-Wall -g -I -l -L`等等，也可以加入`Makefile`文件。

### 版本7：加入其他编译参数
```makefile
src = $(wildcard *.c)
obj = $(patsubst %.c, %.o, $(src))

myArgs = -Wall #-g ...

ALL:test.out

test.out:$(obj)
	gcc $^ -o $@ $(myArgs)

%.o:%.c
	gcc -c $< -o $@ $(myArgs)

clean:
	-rm -rf $(obj) test.out

.PHONY: clean ALL
```

## 作业1 - 复杂一点的项目Makefile
项目目录如下：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_makefile$ tree
.
├── bin
├── inc
│   └── head.h
├── Makefile
├── obj
└── src
    ├── add.c
    ├── div1.c
    ├── main.c
    └── sub.c
```
将源码文件全部放在`src`目录下，`.o`文件放在`obj`目录下，头文件放在`inc`目录下，生成的二进制可执行文件放在`bin`目录下。  
`main.c`：  
```c
#include "head.h"
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
`head.h`：  
```c
#include<stdio.h>
int add(int, int);
int sub(int, int);
int div1(int, int);
```

编写`Makefile`文件，测试通过。   
```makefile
src = $(wildcard ./src/*.c)
obj = $(patsubst ./src/%.c, ./obj/%.o, $(src))

incPath = ./inc
myArgs = -Wall #-g ...

ALL:test.out

test.out:$(obj)
	gcc $^ -o ./bin/$@ $(myArgs)

$(obj):./obj/%.o:./src/%.c
	gcc -c $< -o $@ $(myArgs) -I $(incPath) # 热知识 头文件在预处理阶段展开

clean:
	-rm -rf $(obj) ./bin/test.out

.PHONY: clean ALL
```

## 作业2 - 编译所有C文件
- 使用`make`命令可以将该目录下所有的c文件编译；
- 使用`make xxx`命令可以将该目录下`xxx.c`文件编译为`xxx`可执行文件。

```makefile
src = $(wildcard *.c)
target = $(patsubst %.c, %, $(src)) # 目标是.c文件编译为可执行文件

ALL:$(target)

%:%.c
	gcc $< -o $@

clean:
	-rm -rf $(target)

.PHONY: clean ALL
```
