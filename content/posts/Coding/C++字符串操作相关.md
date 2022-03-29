---
title: C++字符串操作相关
date: 2022-03-29
tags: [C++]
categories: [Coding]
---

## int 和 string 互相转换
```cpp
#include <string>
using namespace std;

int a = stoi("123");
string s = to_string(123);
```

-----

## 按字符分割字符串
使用C库函数`strtok`：  
```cpp
char* strtok( char* str, const char* delim ); // 线程不安全
char *strtok_r(char *str, const char *delim, char **saveptr); // 建议使用
```
参数说明：str为要分解的字符串，delim为分隔符字符串。
返回值：从str开头开始的一个个被分割的串。当没有被分割的串时则返回NULL。
其它：strtok函数线程不安全，可以使用strtok_r替代。
示例：  
```cpp
// 把string data 按照 '_' 分解 存入队列
	char* str = new char[data.size() + 1];
	strcpy(str, data.c_str());
	char* d = "_";
	char* p = strtok(str, d);
	queue<string> dataqueue;
	while (p) {
		string s = p;
		dataqueue.push(s);
		p = strtok(nullptr, d);
	}
```

-----

