---
title: 【MySQL】06 - 单表查询常用操作
date: 2022-10-14
tags: [datebase, MySQL]
categories: [DateBase]
---

## 常见的一些单表查询的操作

- 带`IN`子查询
  - `[NOT]IN(元素1,元素2,...)`
  - `SELECT * FROM user WHERE id IN(1,2,3,4,5,6,7);`
  - `SELECT * FROM user WHERE id NOT IN(1,2,3,4,5,6,7);`
  - 括号中的列表也可以是另一个sql语句
- `DISTINCT`去重
  - 比如，我只想知道有哪些年龄
  - `SELECT DISTINCT age FROM user;`
- `UNION`合并查询
  - 把两个查询语句的结果合并起来，默认去重，`ALL`选项显示重复值
  - `SELECT expression1, expression2,... FROM tables[WHERE conditions] UNION [ALL | DISTINCT] SELECT expression1, expression2,... FROM tables[WHERE conditions];`
  - `SELECT * FROM user WHERE id IN(8,9) UNION SELECT * FROM user WHERE sex='m';`
  - 注意`OR`语句有可能用到索引，因为mysql server有可能会将`OR`查询优化为`UNION`合并查询，并使用索引
- 空值查询
  - `IS [NOT] NULL`
  - `SELECT * FROM user WHERE name IS NULL;`
  - `SELECT * FROM user WHERE name IS NOT NULL;`
- `ORDER BY`排序
  - `SELECT * FROM user ORDER BY age asc;`，升序
  - `SELECT * FROM user ORDER BY age desc;`，降序
- `GROUP BU`分组
  - `SELECT sex FROM user GROUP BY sex;`，按性别分类
  - `SELECT count(id),sex FROM user GROUP BY sex;`，性别分类，每种几个
  - `SELECT COUNT(id),age FROM user GROUP BY age HAVING age>30;`，年龄分类，统计大于30的，每个年龄的有几个
- 分页查询
  - `SELECT * FROM user LIMIT 10;`
  - `SELECT * FROM user LIMIT 2000,10;`