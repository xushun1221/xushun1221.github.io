---
title: 【Linux系统编程】Linux系统知识和基础操作
date: 2022-05-10
tags: [Linux]
categories: [Coding]
---

## 终端
一切输入输出设备的总称，键盘、鼠标、显示器...  
Linux系统中的终端Terminal，是虚拟终端。

## shell
在终端中输入命令会被执行是因为Linux终端中内嵌了shell（命令解释器）。  
查看当前日期时间：`date`  
查看当前系统中有哪些shell：`cat /etc/shells`  
查看当前操作系统正在使用的shell：`echo $SHELL`  
bash是大多数Linux系统的默认shell。

### bash命令和路径补全
bash下输入命令时使用`tab`键可以自动补全，两次`tab`可以把所有开头相同的命令和路径输出。

### history
`history`命令可以查看之前执行过的操作。

### 主键盘快捷键
|操作|快捷键|
|---------|---------|
|上|Ctrl-p|
|下|Ctrl-n|	
|左|Ctrl-b|	
|右|Ctrl-f|	
|Del|Ctrl-d|
|Home|Ctrl-a|
|End|Ctrl-e|
|Backspace|Backspace|
|清除整行|Ctrl-u|
|删除光标到行末| Ctrl-k|
|显示上滚|Shift-PgUp|
|显示下滚|Shift-PgDn|
|增大终端字体|Ctrl-Shift-+|
|减小终端字体|Ctrl- -|
|新打开一个终端|Ctrl-Alt-T|
|清屏|Ctrl-l|

## 类Unix系统的目录结构
最常用命令：
- 查看当前目录：`pwd`  
- 查看当前目录下的文件和目录：`ls`  
- 返回上一级：`cd ..`
- 回到根目录：`cd \`
- 回到home目录：`cd ~`
- 运行当前目录下的可执行文件xx：`./xx`

常用目录：
- bin：存放二进制可执行文件，例如在任意路径下输入`date`命令，shell就会在bin命令下找到`date`命令并执行
- boot：存放开机启动程序
- dev：存放设备文件，Linux中“所见皆文件”，键盘、鼠标等等设备都以文件形式描述，例如查看鼠标设备对应的文件`sudo cat mice`
- home：存放普通用户
- etc：用户信息和系统配置文件 passwd、group
- lib：库文件
- root：管理员宿主目录（家目录），需要先`sudo su`切换为root用户再`cd root`进入root目录
- usr：用户资源管理目录，unix software resource
- media、mnt：和挂载磁盘相关
- proc：和进程相关

## 目录和文件操作
### 绝对路径和相对路径
- 从`/`开始描述的路径为绝对路径，如`/home/xushun`
- 从当前位置开始的路径为相对路径，当前位置可以用`.`表示，上一级目录用`..`表示

### ls
list的简写，列出当前目录下的所有目录和文件。  
常用选项：
- `ls -l`：显示详细信息
- `ls -a`：显示出隐藏文件
- `ll`：显示隐藏文件并显示详细信息
- `ls -d`：显示目录，`ls -dl`显示当前目录的详细信息
- `ls -R`：递归显示该目录下所有文件和目录，有子目录会进入并显示子目录下的内容

### 详细信息说明
|权限|硬链接计数|所有者|所属组|大小|时间|文件名/目录名|
|-----|-----|-----|-----|-----|-----|-----|
|drwxr-xr-x | 4 |xushun |xushun |4096 |4月  |28 20:52 |Downloads|

### 权限说明
|文件类型|所有者的读写执行权限|同组用户的rwx|其他人的rwx|
|-----|-----|-----|-----|
|1|234|567|890|
|d|rwx|r-x|r-x|

### Linux文件类型
共有7/8种：
- 普通文件：-
- 目录文件：d
- 字符设备文件：c
- 块设备文件：b
- 软连接：l
- 管道文件：p
- 套接字：s
- 未知文件（第八种）

在Linux下，不以文件的后缀名来区分文件的类型。

### mkdir
`mkdir dirname`新建目录。

### rmdir
`rmdir dirname`删除空目录，非空的目录删不掉。

### touch
`touch filename`新建空文件。

### rm
- `rm filename`删除文件
- `rm -r dirname`递归删除目录和目录下文件
- `rm -rf dirname`强制删除，不加f也一样

### mv
`mv file1 file2 location`将文件1和文件2移动到目标位置。

### cp
- `cp filename dirname`将文件复制到目录下
- `cp filename1 filename2`复制文件1并重命名为文件2
- `cp -a dir1 dir2`复制目录1及其下所有文件到目录2
- `cp -r dir1 dir2`递归复制目录1到目录2，和`-a`的区别在于`-a`是完全复制，包括文件权限、改动时间、所有者等等。

### cat
- `cat`读终端，回显你在终端种输入的字符
- `cat filename`查看文件内容
- `tac filename`逆序查看文件内容，按行逆序

### more
`more filename`和`cat filename`类似，对于查看大文件更友好，空格翻页，回车下一行，`q`或`ctrl+c`退出。

### less
`less filename`和`cat filename`类似，空格翻页，回车下一行，`q`或`ctrl+c`退出，上和下方向键可以向上或向下翻行，读完不会退出。

### head
`head -n filename`查看文件前n行，不加参数默认为10行。

### tail
`tail -n filename`查看文件后n行，不加参数默认10行，是顺序显示的后n行。

### tree
`tree`显示当前目录结构树，一般不是系统自带，需要先安装。

## 软连接和硬连接
### 软连接
`ln -s file file.s`给文件创建一个软连接`file.s`。  
软连接就像windows系统下的快捷方式，例如查看`file`文件内容就可以通过它的软连接来查看。  
软连接的文件的内容是原文件的访问路径。软连接的权限是软连接本身的权限，而不是原文件的权限。  
如果使用相对路径来创建的软连接，在软连接路径改变之后就无法使用，这一点和Windows的快捷方式不同，快捷方式无论在哪里都可以使用。所以创建软连接应该使用绝对路径。  

### 硬连接
`ln file file.hard`为文件创建一个硬连接。  
每创建一个硬连接，原文件的硬连接计数加一（`ls -l`查看）。对于同一个文件的所有硬连接或文件本身进行的修改，都会同步变化。  
原因是文件和它的硬连接的Inode是相同的，每个文件都有唯一的Inode，直观理解起来就像是C++中的引用，对于同一个文件，无论有多少个引用，在访问时，都是这个文件，修改都是同步的。  
当删除一个硬连接时，文件的硬连接计数会减一，直到计数为0时，才会删除这个文件。即使删除硬连接指向的文件，也只会让硬连接计数减一。

## 用户和用户组
### whoami
`whoami`查看当前用户。

### chmod
change mode，修改文件权限的命令。

#### 文字设定法
`chmod [who] [+|-|=] [mode] filename`，对于文件，给`who`用户添加、删去或赋予`mode`（rwx，读写执行）的权限。  
who：
- `u`：用户，即目录或文件的所有者；
- `g`：同组用户，与文件所有者拥有同组ID的所有用户；
- `o`：其他用户；
- `a`：所有用户，是系统默认值。

[+|-|=]：
- `+`：添加某个权限；
- `-`：删去某个权限；
- `=`：赋予给定权限，并取消其他所有权限（如果有的话）。

#### 数字设定法
`chmod xxx filename`将文件权限设定为`xxx`代表的权限含义。  
读写执行权限的数字表示：
|读 r|写 w|执行 x|
|-----|-----|-----|
|4|2|1|

对于某一类用户的权限是将，这三种权限的数字加起来。例如，文件所有者拥有读写执行rwx权限，同组用户拥有读执行r-x权限，其他用户拥有读权限r--，只需这样设定：`chmod 754 filename`

### su
`su username`切换用户。

### sudo
`sudo [其他命令]`以系统管理员身份执行命令。

### adduser
`sudo adduser newusername`添加新用户。

### deluser
`sudo deluser username`删除用户。

### addgroup
`sudo addgroup newgroupname`创建新组。

### delgroup
`sudo delgroup groupname`删除用户组。

### chown
修改文件的所有者和所属组的命令。
- `sudo chown username filename`将文件`filename`所有者修改为`username`
- `sudo chown username:groupname filename`同时将文件的所有者和所属组修改为`username:groupname`。

### chgrp
`sudo chgrp groupname filename`将文件所属组改为`groupname`。`chown`是更加常用的命令。

## 查找与检索 !重要!
### find
查找文件的命令。
不同查找参数：
- `-type`：按文件类型查找（7种类型，不是后缀名），例如`find ./ -type d`，该命令查找目录文件（没有规定查找深度，默认全部查找）；
- `-name`：按文件名查找，例如`find ./ -name '*.jpg'`，该命令查找所有后缀为`.jpg`的文件；
- `-maxdepth`：指定查找深度，应作为第一个参数使用，例如`find ./ -maxdepth 1 -name '*.jpg'`，该命令表示在根目录下查找所有后缀为`.jpg`的文件，且查找深度为1，即仅在根目录中查找；
- 