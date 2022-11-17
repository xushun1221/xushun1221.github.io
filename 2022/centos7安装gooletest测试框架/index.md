# CentOS7安装GooleTest测试框架


操作系统：CentOS7


## 升级GCC

GoogleTest使用c++14，系统原来的gcc(g++)版本为4.8，需要升级。

1. `sudo yum install centos-release-scl`
2. `sudo yum install devtoolset-8-gcc*`，安装gcc8
3. `scl enable devtoolset-8 bash`，激活对应的devtoolset，可以安装多个版本进行切换
4. 第三点的方式是临时的，重启bash就失效了，使用这条命令长期生效：`echo "source /opt/rh/devtoolset-8/enable" >>/etc/profile`


## 安装CMake

安装版本高点的。

[CentOS7安装CMake](https://xushun1221.github.io/2022/centos7%E5%AE%89%E8%A3%85cmake/)


## 下载GoogleTest安装包

1. `wget https://github.com/google/googletest/archive/refs/tags/release-1.12.1.tar.gz`，[在这找新版本](https://github.com/google/googletest/releases/)
2. `tar zxvf release-1.12.1.tar.gz`，解压
3. `cd googletest-release-1.12.1/`
4. `cmake .`
5. `make`
6. `sudo make install`，安装，头文件在`/usr/local/include/gtest`和`/usr/local/include/gmock`，库文件在`/usr/local/lib64`

## 测试

测试文件

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <gtest/gtest.h>

TEST(COutputPopLimitStrategyTest, PositiveNos) {
    EXPECT_EQ(true,true);
}

int main(int argc,char** argv) {
    ::testing::InitGoogleTest(&argc,argv);
    return RUN_ALL_TESTS();
} 
```

编译运行

```console
[xushun@localhost testgoogletest]$ scl enable devtoolset-8 bash
[xushun@localhost testgoogletest]$ g++ -std=c++11 hello_test.cc -lpthread -lgtest -o
 hello_test
[xushun@localhost testgoogletest]$ ls
hello_test  hello_test.cc
[xushun@localhost testgoogletest]$ ./hello_test 
[==========] Running 1 test from 1 test suite.
[----------] Global test environment set-up.
[----------] 1 test from FunTest
[ RUN      ] FunTest.HandlesZeroInput
[       OK ] FunTest.HandlesZeroInput (0 ms)
[----------] 1 test from FunTest (0 ms total)

[----------] Global test environment tear-down
[==========] 1 test from 1 test suite ran. (0 ms total)
[  PASSED  ] 1 test.
```

注意，编译时要切换到高版本的gcc，因为需要支持c++14特性。
