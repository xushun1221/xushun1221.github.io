# 【VSCode】用户代码片段 - 自动生成代码框架


## 前言
最近写了好多C程序，每次新建一个C文件都要写头文件和main函数，有点烦，想着VSCode能不能自动填写这些东西，还真可以，好耶。

使用 User Snippets 可以通过一个自定义的前缀来自动填写你预设的代码。

## 方法
这里以C为例，其他语言都一样的。

VSCode -> File -> Preferences -> User Snippets -> 选择c -> 打开`c.json`文件（只能在打开C文件的时候使用）

我写的配置：  
```json
{
	// Place your snippets for c here. Each snippet is defined under a snippet name and has a prefix, body and 
	// description. The prefix is what is used to trigger the snippet and the body will be expanded and inserted. Possible variables are:
	// $1, $2 for tab stops, $0 for the final cursor position, and ${1:label}, ${2:another} for placeholders. Placeholders with the 
	// same ids are connected.
	// Example:
	// "Print to console": {
	// 	"prefix": "log",
	// 	"body": [
	// 		"console.log('$1');",
	// 		"$2"
	// 	],
	// 	"description": "Log output to console"
	// }
	"HEADER": {
		"prefix": "header",
		"description": "为c程序添加常用头文件和main函数",
		"body": [
			"/*",
			"@Filename : $TM_FILENAME", 
			"@Description : ", 
			"@Datatime : $CURRENT_YEAR/$CURRENT_MONTH/$CURRENT_DATE $CURRENT_HOUR:$CURRENT_MINUTE:$CURRENT_SECOND", 
			"@Author : xushun", 
			"*/",
			"#include <unistd.h>",
			"#include <stdio.h>",
			"#include <stdlib.h>",
			"#include <string.h>",
			"#include <errno.h>",
			"#include <pthread.h>",
			"",
			"int main(int argc, char** argv) {",
			"\t$0",
			"\treturn 0;",
			"}"
		],
	}
}
```

解释一下，你可以自定义用户代码片段，我这里自定义了一个`"HEADER"`，每个代码片段都有三项内容：
- `"prefix"`：你输入这个前缀然后按tab或回车就可以自动替换为`"body"`中的内容了，例如，我在C文件中输入`header`，回车，就会自动填写一些常用头文件和main函数；
- `"description"`：描述这个代码片段；
- `"body"`：按行描述的代码片段的具体内容，注意，`$1`表示，自动填充后光标出现的第一个位置，填写完这个位置的内容后，按tab，就可以跳到`$2`的位置，...，所有的`$数字`位置都填完后，按tab，就会跳到`$0`的位置，结束输入。
