---
title: 'MySQL技术内幕：SQL编程——数据类型小结'
layout: post
categories: 技术
tags:
    - MySQL
---

## 类型属性 ##

### UNSIGNED ###

unsigned属性会将数字类型无符号化。但是在mysql中，对于unsigned数的操作，其返回值都是unsigned的，因此可能会有减法溢出的情况。如下面例子。

```
create table t (a int UNSIGNED, b int UNSIGNED) engine=innodb;

insert into t select 1, 2;

select a - b from t;
```
select a - b的计算结果是不确定，在不同的系统上会有不同的答案。可能是-1，也可能是一个很大的正值。

### ZEROFILL ###

zerofill属性影响显示，但不会改变实际存储的值。

```
mysql> show create table t;
CREATE TABLE `t` (
  `a` int(10) UNSIGNED DEFAULT NULL,
  `b` int(10) UNSIGNED DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

int(10)中的10，如果没有zerofill属性，就毫无意义。

```
mysql> select * from table t;
+------+------+
| a    | b    |
+------+------+
|    1 |    2 |
+------+------+
```

对a列增加zerofill属性：

```
alter table t change column a a int(4) UNSIGNED ZEROFILL;
```

再次查询：

```
mysql> select * from t;
+------+------+
| a    | b    |
+------+------+
| 0001 |    2 |
+------+------+
```

这就是zerofill的作用，如果宽度小于设定的宽度（这里是4），则自动填充0。注意，这只是显示的效果，实际存储还是1。

## 日期和时间类型 ##

### DATETIME ###

- 占用8字节
- 范围为：1000-01-01 00:00:00 ~ 9999-12-31 23:59:59

### DATE ###

- 占用3字节
- 范围为：1000-01-01 ~ 9999-12-31

### TIMESTAMP ###

- TIMESTAMP和DATETIME显示结果一样，即YYYY-MM-DD HH:MM:SS的形式
- TIMESTAMP占用4个字节
- 范围为：1970-01-01 00:00:00 UTC ~ 2038-01-19 03:14:07 UTC
- 实际存储的内容为1970-01-01 00:00:00到当前时间的毫秒数
- 建表时，TIMESTAMP的日期类型可以设置一个默认值，而DATETIME不行
- 在更新表是，可以设置TIMESTAMP类型的列自动更新为当前时间

### YEAR ###

- 占用1字节
- 在定义时，可以指定显示的宽度为YEAR(4)或YEAR(2)
- 对于YEAR(4)，显示的年份范围为：1901 ~ 2155；对于YEAR(2)，显示的年份范围为：1970 ~ 2070，其中00 ~ 69代表2000 ~ 2069

### TIME ###

- 占用3字节
- 范围为：-838:59:59 ~ 838:59:59
- TIME类型可以大于23，是因为TIME不仅可以保存一天中的时间，也可以用于保存时间间隔，这也是为什么TIME会有负值的原因

### 与日期时间相关的函数 ###

NOW、CURRENT_TIMESTAMP和SYSDATE

- CURRENT_TIMESTAMP是NOW的同义词，二者相等
- SYSDATE函数返回的是执行到当前函数时的时间，而NOW函数返回的是执行SQL语句时的时间

时间加减函数DATE_ADD和DATE_SUB

- 函数声明：DATE_ADD(date, INTERVAL expr unit)、DATE_SUB(date, INTERVAL expr unit)
- expr可以是正值，也可以是负值，因此可以使用DATE_ADD实现DATE_SUB
- 如果目标日期是闰月，则返回日期为2月29日，否则为2月28日

格式化函数DATE_FORMAT

```
mysql> select date_format(now(), '%Y%m%d') as datetime;
+----------+
| datetime |
+----------+
| 20131009 |
+----------+
```

注意不要错误的使用该函数：
    
```
select * from table where date_format(date, '%Y%m%d')='xxxx-xx-xx';
```

如果对日期类型有索引，使用上述语句时，索引会失效，执行效率可能会非常低。

## 数字类型 ##

### 整型 ###

<table>
    <tr>
        <td>类型</td>
        <td>占用空间（字节）</td>
        <td>最小值（signed / UNSIGNED）</td>
        <td>最大值（signed / UNSIGNED）</td>
    </tr>
    <tr>
        <td>TINYINT</td>
        <td>1</td>
        <td>-128 / 0</td>
        <td>127 / 255</td>
    </tr>
    <tr>
        <td>SMALLINT</td>
        <td>2</td>
        <td>-32 768 / 0</td>
        <td>32 767 / 65 535</td>
    </tr>
    <tr>
        <td>MEDIUMINT</td>
        <td>3</td>
        <td>-83 88 608 / 0</td>
        <td>83 88 607 / 16 777 215</td>
    </tr>
    <tr>
        <td>INT</td>
        <td>4</td>
        <td>-2 147 483 648 / 0</td>
        <td>2 147 483 647 / 4 294 967 295</td>
    </tr>
    <tr>
        <td>BIGINT</td>
        <td>8</td>
        <td>-9 223 372 036 854 775 808 / 0</td>
        <td>9 223 372 036 854 775 807 / 18 446 744 073 709 551 615</td>
    </tr>
</table>

可以使用ZEROFILL属性格式化显示整型，但一旦启动ZEROFILL属性，MySQL会自动为列添加UNSIGNED属性。

### 浮点型 ###

单精度FLOAT类型、双精度DOUBLE PRECISION类型，两种类型都是非精确的类型，经过计算后并不能保证运算的正确性，如M * G /G 不一定等于M。

### 高精度类型 ###

DECIMAL和NUMERIC类型在MySQL中视为相同的类型，用于保存必须为确定精度的值。使用时通常必须指定精度和标度，如salary DECIMAL(5, 2)中，5是精度，2是标度。精度表示保存值的主要位数，标度表示小数点后可以保存的位数。

### 位类型 ###

位类型，即BIT数据类型可以用来保存位字段的值。BIT(M)表示允许存储M位数的值，M范围为1到64，占用的空间为(M + 7) / 8字节。如果为BIT(M)分配的值的长度小于M，则左侧用0补齐。select一个BIT类型的值，会出现空结果的情况，这时需要HEX函数，即select HEX(a) from t。

## 字符型 ##

### 字符集 ###

MySQL默认的字符集是latin1，为了国际化需求，推荐使用utf-8。

### 排序规则 ###

排序规则是指对指定字符集下不同字符的排序规则。其特征有：

- 两个不同的字符集不能有相同的排序规则
- 每个字符集有一个默认的排序规则
- 有一些常用的命名规则，如_ci结尾表示大小写不敏感，_cs结尾表示大小写敏感，_bin结尾表示二进制比较

utf-8默认的排序规则是utf8_general_ci，因此a和A视为同一个字符，如：

```
mysql> select 'a'='A';
+---------+
| 'a'='A' |
+---------+
|       1 |
+---------+
```

对于创建的表t，如果对a列需要区分大小写，可以将a列的排序规则改为utf8_bin，如：

```
ALTER TABLE t MODIFY COLUMN a VARCHAR(10) COLLATE UTF8_BIN;
```

### CHAR和VARCHAR ###

CHAR和VARCHAR是两种最常用的字符类型。CHAR(N)用来保存固定长度的字符，VARCHAR(N)用来保存变长字符类型。对于CHAR类型，N的范围为0到255，对于VARCHAR类型，N的范围为0到65535。N表示字符长度，而不是字节长度。

对于CHAR类型，MySQL会自动对存储列的右边进行填充，直到字符串达到指定的长度N。而在读的时候自动将填充的字符删除。除非指定SQL_MODE为PAD_CHAR_TO_FULL_LENGHT。

与CHAR类型不同的是，VARCHAR存储是需要在前缀长度列表加上实际存储的字符，该字符占用1到2个字节。当存储的字符串长度小于255字节时，需要1个字节的空间，当大于255时，需要2个字节的空间。

虽然CHAR和VARCHAR存储方式不同，但二者进行比较时，只比较其值，而忽略CHAR右边的填充。

### BINARY和VARBINARY ###

BINARY和VARBINARY类似于CHAR和VARCHAR，但存储的是二进制的字符串，而不是字符型的字符串。也就是说BINARY和VARBINARY没有字符集，对其排序和比较都是按照二进制进行。

- BINARY(N)和VARBINARY(N)中的N指的是字节长度，而不是CHAR(N)和VARCHAR(N)中的字符长度
- CHAR和VARCHAR进行字符比较时，比较的是存储的字符本身，忽略填充字符；而BINARY和VARBINARY则按照二进制比较，会考虑到填充字符，及'a'与'a  '不同
- BINARY的填充字符时0x00，而CHAR的填充字符时0x20