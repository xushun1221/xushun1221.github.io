# 【C++基础】03 - 从编译器角度理解C++代码的编译和链接原理


从一个简单的例子入手，分析C\C++编译链接的过程。

## 示例
`03main.cpp`：  
```C++
extern int gdata;
int sum(int, int);

int data = 20;

int main() {
    int a = gdata;
    int b = data;
    int ret = sum(a, b);
    return 0;
}
```

`03sum.cpp`：  
```C++
int gdata = 10;
int sum(int a, int b) {
    return a + b;
}
```

## 分析编译链接过程

等我看完自我修养再更新。
