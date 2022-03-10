---
title: Coding之小技巧
date: 2022-02-16
tags: [算法与数据结构, C++]
categories: [Coding]
---

## 前言
记录一些Coding的技巧。

## 取数组下标中点 22.2.16
常规的取下标中点方法：`mid = (left + right) / 2`，这种方法在数组很大的时候，可能会溢出，所以用这种方法：`mid = left + (right - left) / 2`，不溢出的原因是：  
`left`、`right`本身不溢出，且`right >= left`，`(right - left) / 2`肯定不溢出。  
更简化一点的写法：`mid = left + ((right - left) >> 1)`。*右移操作比除法快*

-----

## master公式
计算子问题等规模的递归问题的时间复杂度。  
- 如果一个递归过程满足：$T\left(N\right)=aT\left(\frac{N}{b}\right)+O\left(N^{d}\right)$
- 如果：$log_{b}a<d$，那么时间复杂度为：$O\left(N^{d}\right)$
- 如果：$log_{b}a>d$，那么时间复杂度为：$O\left(N^{log_{b}a}\right)$
- 如果：$log_{b}a=d$，那么时间复杂度为：$O\left(N^{d}log_{2}N\right)$

-----

## C++ for循环continue问题
在C++的for循环中，`continue`语句并不能跳过步进语句。  
例子：  
```cpp
for (int i = 0; i < 10; ++ i)
{
	cout << "i" << endl;
	continue;
}
```
这段代码并不会进入死循环，每轮输出之后，`continue`语句不会跳过`++ i`。

-----

## C/C++ size_t
`size_t`是一些C/C++标准在`stddef.h`中定义的，`size_t`类型表示C中任何对象所能达到的最大长度，它是无符号整数。  
它是为了方便系统之间的移植而定义的，不同的系统上，定义`size_t`可能不一样。`size_t`在32位系统上定义为 `unsigned int`，也就是32位无符号整型。在64位系统上定义为 `unsigned long` ，也就是64位无符号整形。`size_t` 的目的是提供一种可移植的方法来声明与系统中可寻址的内存区域一致的长度。

-----

## std::map之iterator和erase问题
在std::map中可以使用erase方法来删掉某一个节点，但是在循环里使用的话就会出现问题，会导致程序的行为不可知，因为map是关联容器，对于关联容器而言，如果一个元素被删除，那么对应的迭代器就失效了。正确的用法应该是，使用erase返回的迭代器（指向下一个元素，C++11）。  
```cpp
iter = testmap.erase(iter);
```
或者删除之前，定位到下一个元素。（STL推荐）  
```cpp
testmap.erase(iter ++);
```