---
title: 'MySQL技术内幕：SQL编程——查询处理'
layout: post
categories: 技术
tags:
    - MySQL
---

## 逻辑查询处理 ##

```
(8) select (9) distinct <select_list>
(1) from <left_table>
(3) <join_type> join <right_table>
(2) on <join_condition>
(4) where <where_condition>
(5) group by <group_by_list>
(6) with (cube | rollup)
(7) having <having_condition>
(10) order by <order_by_list>
(11) limit <limit_number>
```

上例中，一共11个步骤，最先执行from，最后执行limit。每个操作都会产生一张虚拟表，该虚拟表作为一个处理的输入。这些虚拟表是透明的，只有最后一个虚拟表才会返回。

1. from：对from中的左表<left_table>和右表<right_table>执行笛卡尔积，产生虚拟表VT1；
2. on：对虚拟表VT1应用on筛选，只有符合<join_condition>的才会插入到虚拟表VT2；
3. join：如果指定了outer join（left outer join、right outer join），那么保留表中未匹配的行作为外部行添加到虚拟表VT2中，产生虚拟表VT3；如果from中包含两个以上的表，则对上一个连接生成的结果表VT3和下一个表重复1~3步，直到处理完所有的表；
4. where：对虚拟表VT3应用where筛选条件，只有符合<where_condition>的才会插入到虚拟表VT4中；
5. group by：根据group by字句中的列，对VT4中的记录进行分组操作，产生虚拟表VT5；
6. with：对虚拟表VT5进行cube或rollup操作，产生虚拟表VT6;
7. having：对虚拟表VT6进行having条件过滤，只有符合<having_condition>的记录才会插入到虚拟表VT7中；
8. select：执行select操作，选择指定的列，插入到虚拟表VT8中；
9. distinct：去除重复数据，产生虚拟表VT9；
10. order by：将虚拟表VT9中的数据按照<order_by_list>进行排序操作，产生虚拟表VT10；
11. limit：取出指定行数的记录，产生VT11，并返回。