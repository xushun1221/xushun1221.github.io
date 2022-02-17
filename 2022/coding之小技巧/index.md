# Coding之小技巧


## 前言
记录一些Coding的技巧。

## 取数组下标中点 22.2.16
常规的取下标中点方法：`mid = (left + right) / 2`，这种方法在数组很大的时候，可能会溢出，所以用这种方法：`mid = left + (right - left) / 2`，不溢出的原因是：  
`left`、`right`本身不溢出，且`right >= left`，`(right - left) / 2`肯定不溢出。  
更简化一点的写法：`mid = left + ((right - left) >> 1)`。*右移操作比除法快*

-----

## master公式 22.2.16
计算子问题等规模的递归问题的时间复杂度。  
- 如果一个递归过程满足：$T\left(N\right)=aT\left(\frac{N}{b}\right)+O\left(N^{d}\right)$
- 如果：$log_{b}a<d$，那么时间复杂度为：$O\left(N^{d}\right)$
- 如果：$log_{b}a>d$，那么时间复杂度为：$O\left(N^{log_{b}a}\right)$
- 如果：$log_{b}a=d$，那么时间复杂度为：$O\left(N^{d}log_{2}N\right)$

-----

## C++ for循环continue问题 22.2.17
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



