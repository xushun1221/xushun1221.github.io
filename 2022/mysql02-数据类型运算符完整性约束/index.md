# 【MySQL】02 - 数据类型、运算符、完整性约束


## 数据类型

MySQL数据类型定义了数据的大小范围，因此使用时选择合适的类型，不仅会降低表占用的磁盘空间，间接减少了磁盘I/O的次数，提高了表的访问效率，而且索引的效率也和数据的类型息息相关。

### 数值类型

![](/post_images/posts/Database/MySQL/数值类型.jpg "数值类型")

> 精确计算时，浮点类型推荐使用`decimal`类型，超出范围会保存，`FLOAT`和`DOUBLE`类型超出范围不会报错，直接截断。

> `age INT(4)`，这里的4不是内存大小，只表示数据显示的宽度。

### 字符串类型

![](/post_images/posts/Database/MySQL/字符串类型.jpg "字符串类型")

> `CHAR(12)`和`VARCHAR(12)`的区别在于，`CHAR`长度固定而`VARCHAR`长度可变，如`'hello'`字符串使用5个字节`CHAR(12)`仍然使用12字节，而`VARCHAR(12)`使用5个字节。  
> 超出12字节的字符串都被截断。

> `BLOB`等类型存储二进制文件  
> `TEXT`等类型存储文本文件

> 字符串使用单引号括起。

### 时间和日期类型

![](/post_images/posts/Database/MySQL/时间和日期类型.jpg "时间和日期类型")


> 日期类型也是做项目过程中，经常使用的类型信息，尤其是`TIMESTAMP`和`DATETIME`两个类型，但是注意`TIMESTAMP`会自动更新时间，非常适合那些需要记录最新更新时间的场景，而`DATETIME`需要手动更新。


### ENUM和SET

> 这两个类型，都是限制该字段只能取固定的值，但是枚举字段只能取一个唯一的值，而集合字段可以取多个值。



## 运算符

### 算术运算符

![](/post_images/posts/Database/MySQL/算术运算符.jpg "算术运算符")

> 注意数据运算后的结果是否会越界。

### 逻辑运算符

![](/post_images/posts/Database/MySQL/逻辑运算符.jpg "逻辑运算符")

### 比较运算符

![](/post_images/posts/Database/MySQL/比较运算符.jpg "比较运算符")





## 完整性约束

数据完整性约束是在表和字段上强制执行的数据检验规则，为了防止不规范的数据进入数据库，在用户对数据进行插入、修改、删除等操作时，DBMS自动按照一定的约束条件对数据进行监测，主要是对空值和重复值的约束，使不符合规范的数据不能进入数据库，以保证数据存储的完整性和准确性。

示例：  
```sql
CREATE TABLE user(
    id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT COMMENT '用户主键id 自增',
    nickname VARCHAR(50) UNIQUE NOT NULL COMMENT '用户的昵称 非空 唯一键',
    age TINYINT UNSIGNED NOT NULL DEFAULT 18 COMMENT '用户年龄 非空 默认18',
    sex ENUM('male','female')
);
```
按照约束的不同类型可以分为实体完整性（PRIMARY KEY、UNIQUE、AUTO_INCREMENT）、域完整性（NOT NULL、DEFAULT）、参照完整性（FOREIGN KEY）、用户自定义完整性（CHECK）。

1. 主键约束 PRIMARY KEY
   - 每个表中只能有一个主键
   - 主键值非空不重复
   - 可设置单字段主键，也可设置多字段联合主键
   - 联合主键中多个字段的数据完全相同时，才违反主键约束
2. 自增键约束 AUTO_INCREMENT
   - 指定字段的数据自动增长
   - 配合主键一起使用，并且只适用于整数类型
   - 默认从1开始，每增加一条记录，该字段的值会增加1
   - 即使数据删除，还是从删除的序号继续增长
3. 唯一键约束 UNIQUE
   - 指定列的数据不能重复
   - 可以为空，但只能出现一个空值
4. 非空约束 NOT NULL
   - 字段的值不能为空
5. 默认值约束 DEFAULT
   - 如果新插入一条记录时没有为该字段赋值，系统会自动为这个字段赋值为默认约束设定的值
   - 如果插入的数据为`null`，则不会使用默认值，只有没有插入数据时候，才会使用默认值
6. 外键约束 FOREIGN KEY
   - 某一表中某字段的值依赖于另一张表中某字段的值
   - 主键所在的表为主表，外键所在的表为从表
   - 每一个外键值必须与另一个表中的主键值相对应


> 注意，在现代的后端开发中，我们通常不使用外键、存储函数、存储过程、触发器等逻辑，因为这些逻辑通常是由mysql本身控制的。我们应该将这些功能在业务层代码中实现，而非交由mysql实现，以减少mysql的压力。


