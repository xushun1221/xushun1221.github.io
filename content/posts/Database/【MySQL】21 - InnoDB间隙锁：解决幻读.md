---
title: 【MySQL】21 - InnoDB间隙锁：解决幻读
date: 2022-11-09
tags: [datebase, MySQL]
categories: [DateBase]
---

串行化隔离级别下，如何解决虚读（幻读）问题？

答：间隙锁（gap lock）。


## 幻读问题

回顾一下幻读问题。

<html>
    <table style="margin: auto">
        <tr>
            <td valign="top">A事务<br>
                <!--左侧内容-->
                <pre><code>
begin;


insert into xxx_table values(满足指定条件);
commit;
                </code></pre>
            </td>
            <td valign="top">B事务<br>
                <!--右侧内容-->
                <pre><code>
begin;
select * from xxx_table where 指定条件;
(原始数据)

select * from xxx_table where 指定条件;
(出现一行新数据)
                </code></pre>
            </td>
        </tr>
    </table>
</html>