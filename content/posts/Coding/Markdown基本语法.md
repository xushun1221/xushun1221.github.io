---
title: Markdown基本语法
date: 2022-01-17
tags: [markdown]
categories: [Coding]
---

## 更新注意事项
- hugo搭建blog，使用markdown中代码块时，需要使用`LF`换行模式，否则会出错；
- 表格不能光有表头。

## 1. 标题

在单词短句之前添加井号`#`来创建标题，井号的数量代表标题的级别，一到六个`#`代表一级到六级标题。<br>*注意，最好在`#`后标题内容之前加上一个空格。*

例如，可使用下列语句创建一个三级标题。
```markdown
### 三级标题
```

另外，一级、二级标题可以在标题内容下分别使用若干`=`和`-`来表示。
```markdown
一级标题
=======

二级标题
-------
```


## 2. 段落

使用空白的一行将两个段落分隔开。
```markdown
一个段落

下一个段落
```

*注意，不要再段落开头使用空格或制表符来缩进段落*


## 3. 换行

在一行的末尾加两个或多个空格，然后按回车键，即可另起一行。
```markdown
这是第一行，后面添加两个空格  
这是第二行
```

还可以通过HTML的`<br>`标签来换行，只需在一行的末尾添加`<br>`，接着输入的就是下一行的内容。
```markdown
这是第一行<br>这是第二行
```


## 4. 强调

加粗文本，需要在一段文本的前后各添加两个星号`*`或下划线`_`。
```markdown
用**星号**来加粗

用__下划线__来加粗
```
*注意：建议使用星号*

斜体显示，需要在一段文本的前后各添加一个星号`*`或下划线`_`。
```markdown
*用星号斜体一句话*

用*星号*来斜体一个单词

_用下划线来斜体一句话_
```
*注意：如果在文本中使用斜体，请在单词前后使用一个星号，且不要用空格*

同时加粗和斜体，需要在一段文本前后各添加三个星号`*`或下划线`_`。
```markdown
***用三个星号加粗并斜体一句话***

用三个***星号***加粗并斜体一个单词
```
*注意：建议使用三个星号来加粗并斜体*


## 5. 引用

在段落前添加一个大于号`>`来创建一个块引用，`>`后可加可不加空格。
```markdown
> 这是一个引用
```

块引用包含多个段落时，为段落之间的空白行添加一个`>`。
```markdown
> 这是第一段
>
> 这是第二段
```

块引用可以嵌套，需要在要嵌套的段落前加两个大于号`>>`。
```markdown
> 这是第一段
>
>> 嵌套引用的一段
```

块引用可以包含其他的Markdown格式的元素，但不是所有元素都可以使用。
```markdown
> ### 标题
> 
> - 列表
> - 列表
> 
> *斜体* 
```


## 6. 列表

有序列表，在每个表项前添加数字并紧跟一个英文句点`.`即可。数字不必按自然数排序，但应当以数字一`1`开始，缩进表项即可嵌套列表。
```markdown
1. xx
2. xx
3. xx
4. xx

1. xx
1. xx
1. xx
1. xx

1. xx
2. xx
3. xx
	1. xx
	2. xx
4. xx
```

无序列表，在每个表项前添加破折号`-`、星号`*`或者加号`+`，不应混合使用多种符号创建无序列表。
```markdown
- xx
- xx
- xx

* xx
* xx
* xx

+ xx
+ xx
+ xx

- xx
- xx
	- xx
	- xx
- xx
```

在列表中嵌套其他元素，只需将该元素缩进四个空格或一个制表符。
```markdown
- xx
- xx
	其他元素
- xx
```


## 7. 代码

将单词或短语表示为代码，需要用两个反引号包裹`` ` ``。
```markdown
这是一个`代码`写法
```

如果代码中有一个或多个反引号，将代码包裹在双反引号中。
```markdown
``在段落中使用`代码`来表示``
```

要创建代码块，只需使用一对三个反引号，将代码块包围，在第一个三反引号后标注代码的语言。<br>或者直接将代码片段整体缩进四个空格或一个制表符。
```markdown
(三个`)cpp
	#include <iostream>
	using namespace std;
	int main(int argc; char ** argv) {
		cout << "test" << endl;
		return 0;
	}
(三个`)
```

## 8. 分隔线

创建分割线，在单独的一行上，使用三个或多个星号`***`、破折号`---`或下划线`___`，并且不能包括其他内容。
```markdown

***

---

___

```
*注意：需要在分隔线前后均添加一个空白行*


## 9. 链接

链接，使用一对中括号`[]`和一对括号`()`表示，链接文本在中括号中，地址在括号中。
```markdown
这是我的博客地址[XuShun's Blog](https://xushun1221.github.io)
```

链接可以添加title，就是鼠标悬停时出现的文字，用双引号括起放在链接后面。
```markdown
这是我的博客地址[XuShun's Blog](https://xushun1221.github.io "我的博客")
```

网址和Email可以方便的用`<>`括起，就变成可以点击的链接了。
```markdown
<https://xushun1221.github.io>
<fake@example.com>
```

强调的链接，在链接前后加上星号`*`，可以加粗或是斜体。
```markdown
这是我的加粗博客地址**[XuShun's Blog](https://xushun1221.github.io)**
这是我的斜体博客地址*[XuShun's Blog](https://xushun1221.github.io)*
```

将链接表示为代码，只要用反引号`` ` ``将链接文本括起。
```markdown
这是我的代码博客地址[`XuShun's Blog`](https://xushun1221.github.io)
```


## 10. 图片

使用感叹号`!`添加图片，方括号添加替代文本，图片链接放在括号中，链接后可以增加一个可选的图片标题。
```markdown
![这是图片](/assets/img/xxx.jpg "图片")
```

给图片添加链接，将图像Markdown括在方括号中，链接添加在圆括号中。
```markdown
[![这是图片](/assets/img/xxx.jpg "图片")](https://xxxxx.xxx)
```


## 11. 转义字符

显示用于格式化Markdown文档的字符，在字符前添加反斜杠`\`。
```markdown
\* 显示星号的写法
```

可以转义的字符：
- \
- \`
- *
- _
- {}
- []
- ()
- \#
- +
- -
- .
- !
- |

