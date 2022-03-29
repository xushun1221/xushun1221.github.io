---
title: C++输入输出
date: 2022-03-25
tags: [C++]
categories: [Coding]
---

## 基本输入
`cin >>`会自动过滤掉：空格、回车、TAB等不可见字符（会被吃掉，不会保留在输入流中），可以用于`string`和数组

`cin.get(字符变量)`用来接收一个字符，可以接收空格，遇到回车时终止
`cin.get(数组名arr, 接收字符数目n)`最后一个字符会是`\0`，如果输入字符数`>= n`，只会接收`n - 1`个字符，后面自动添加`\0`，遇到回车会直接终止输入
`cin.get(void)`用来舍去（吃掉）输入流中后面一个字符

`cin.getline(数组名arr, 接收字符数目n, 结束字符)`默认结束字符是`\0`，实际输入字符比`n`少一个，因为最后一个字符是结束字符

`getline(cin, string)`接收的是`string`而不是数组，可以接收空格和回车，结尾不是`\0`

-----

## cout控制浮点数小数位数 （四舍五入）
```cpp
#includ <iomanip>
cout << setiosflags(ios::fixed); 		//只有在这项设置后，setprecision才是设置小数的位数。
cout << setprecision(0) << f << endl; 	//输出0位小数，3
```

-----

## 浮点数四舍五入保留整数部分
```cpp
int(0.5 + double(num))
```

-----

