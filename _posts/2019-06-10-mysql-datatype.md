---
layout: post
title: MySQL数据类型（part1）
date: 2019-06-10
categories:
    - MySQL
comments: true
permalink: mysql-datatype.html
---

# 1. 各数据类型的取值范围 

- **TINYINT** -128 - 127 
- **TINYINT UNSIGNED** 0 - 255 
- **SMALLINT** -32768 - 32767 
- **SMALLINT UNSIGNED** 0 - 65535 
- **MEDIUMINT** -8388608 - 8388607 
- **MEDIUMINT UNSIGNED** 0 - 16777215 
- **INT 或 INTEGER** -2147483648 - 2147483647 
- **INT UNSIGNED 或 INTEGER UNSIGNED** 0 - 4294967295 
- **BIGINT** -9223372036854775808 - 9223372036854775807 
- **BIGINT UNSIGNED** 0 - 18446744073709551615 
- **FLOAT** -3.402823466E+38 - -1.175494351E-38,0,1.175494351E-38 - 3.402823466E+38 
- **DOUBLE 或 DOUBLE PRECISION 或 REAL** -1.7976931348623157E+308 - -2.2250738585072014E-308,0,2.2250738585072014E-308 - 1.7976931348623157E+308 
- **DECIMAL[(M,[D])] 或 NUMERIC(M,D)** 由M(整个数字的长度,包括小数点,小数点左边的位数,小数点右边的位数,但不包括负号)和D(小数点右边的位数)来决定,M缺省为10,D缺省为0 
- **DATE** 1000-01-01 - 9999-12-31 
- **DATETIME** ’1000-01-01 00:00:00.000000’ 到 ‘9999-12-31 23:59:59.999999’
- **TIMESTAMP** ’1970-01-01 00:00:01.000000’ 到 ‘2038-01-19 03:14:07.999999’
- **TIME**-838:59:59' to 838:59:59 
- **`YEAR[(2|4)]`** 缺省为4位格式,4位格式取值范围为1901 - 2155,0000,2位格式取值范围为70-69(1970-2069) 
- **CHAR(M) [BINARY] 或 NCHAR(M) [BINARY]** M的范围为1 - 255,如果没有BINARY项,则不分大小写,NCHAR表示使用缺省的字符集.在数据库中以空格补足,但在取出来时末尾的空格将自动去掉. 
- **[NATIONAL] VARCHAR(M) [BINARY]** M的范围为1 - 255.在数据库中末尾的空格将自动去掉. 
- **TINYBLOB 或 TINYTEXT** 255(2^8-1)个字符 
- **BLOB 或 TEXT** 65535(2^16-1)个字符 
- **MEDIUMBLOB 或 MEDIUMTEXT** 16777215 (2^24-1)个字符 
- **LONGBLOB 或 LONGTEXT** 4294967295 (2^32-1)个字符 
- **ENUM('value1','value2',...)** 可以总共有65535个不同的值 ，MySQL 后台存储以下标的方式，也就是 tinyint 或者 smallint 的方式，下标从 1 开始。
- **SET('value1','value2',...)** 最多有64个成员 

# 几种整数类型

**1、bigint**

从 -2^63 (-9223372036854775808) 到 2^63-1 (9223372036854775807) 的整型数据（所有数字），无符号的范围是0到

18446744073709551615。一位为 8 个字节。

**2、int**

一个正常大小整数。有符号的范围是-2^31 (-2,147,483,648) 到 2^31 - 1 (2,147,483,647) 的整型数据（所有数字），无符号的范围是0到4294967295。一位大小为 4 个字节。
**int** 的 SQL-92 同义词为 **integer**。

**3、mediumint**
一个中等大小整数，有符号的范围是-8388608到8388607，无符号的范围是0到16777215。 一位大小为3个字节。

**4、smallint**

一个小整数。有符号的范围是-2^15 (-32,768) 到 2^15 - 1 (32,767)  的整型数据，无符号的范围是0到65535。一位大小为 2  个字节。MySQL提供的功能已经绰绰有余，而且由于MySQL是开放源码软件，因此可以大大降低总体拥有成本。

**5、tinyint**

有符号的范围是-128 - 127，无符号的范围是 从 0 到 255 的整型数据。一位大小为 1 字节。

**int(M) 在 integer 数据类型中,M 表示最大显示宽度。在 int(M) 中,M 的值跟 int(M) 所占多少存储空间并无任何关系。和数字位数也无关系 int(3)、int(4)、int(8) 在磁盘上都是占用 4 btyes 的存储空间。**

>  注意，所有算术运算用有符号的BIGINT或DOUBLE值完成，因此你不应该使用大于9223372036854775807（63位)的有符号大整数，除了位函数！注意，当两个参数是INTEGER值时，-、+和*将使用BIGINT运算！这意味着如果你乘2个大整数(或来自于返回整数的函数)，如果结果大于9223372036854775807，你可以得到意外的结果。一个浮点数字，不能是无符号的，对一个单精度浮点数，其精度可以是<=24，对一个双精度浮点数，是在25   和53之间，这些类型如FLOAT和DOUBLE类型马上在下面描述。FLOAT(X)有对应的FLOAT和DOUBLE相同的范围，但是显示尺寸和小数位数是未定义的。在MySQL3.23中，这是一个真正的浮点值。在更早的MySQL版本中，FLOAT(precision)总是有2位小数。该句法为了ODBC兼容性而提供。



# Char和Varchar的区别



# 日期类型

日期类型包含了 date,time,datetime,timestamp，以及 year。year 占 1 Byte，date 占 3 Byte。　

 time,timestamp,datetime 在不包含小数位时分别占用 3 Byte,4 Byte,8 Byte；小数位部分另外计算磁盘占用，

| 小数位精度 | 占用字节 |
| ---------- | -------- |
| 0          | 0        |
| 1,2        | 1        |
| 3,4        | 2        |
| 5,6        | 3        |

**timestamp与datetime的区别**

DATETIME的默认值为null；TIMESTAMP的字段默认不为空（not null）,默认值为当前时间（CURRENT_TIMESTAMP），如果不做特殊处理，并且update语句中没有指定该列的更新值，则默认更新为当前时间。

>  如果我们用timestamp，我们可以不管这个字段就能自动更新了如果用datetime，不会有自动更新当前时间的机制，所以需要在上层手动更新该字段

DATETIME使用8字节的存储空间，TIMESTAMP的存储空间为4字节（ 代表的时间戳是一个 int32 存储的整数）。因此，TIMESTAMP比DATETIME的空间利用率更高。

两者的存储方式不一样 ，对于TIMESTAMP，它把客户端插入的时间从当前时区转化为UTC（世界标准时间）进行存储。查询时，将其又转化为客户端当前时区进行返回。而对于DATETIME，不做任何改变，基本上是原样输入和输出

两者所能存储的时间范围不一样 

- timestamp所能存储的时间范围为：’1970-01-01 00:00:01.000000’ 到 ‘2038-01-19 03:14:07.999999’；
- datetime所能存储的时间范围为：’1000-01-01 00:00:00.000000’ 到 ‘9999-12-31 23:59:59.999999’。

综上所述，日期这块类型的选择遵循以下原则：

1. 如果时间有可能超过时间戳范围，优先选择 datetime。

2. 如果需要单独获取年份值，比如按照年来分区，按照年来检索等，最好在表中添加一个 year 类型来参与。

3. 如果需要单独获取日期或者时间，最好是单独存放，而不是简单的用 datetime 或者 timestamp。后面检索时，再加函数过滤，以免后期增加 SQL 编写带来额外消耗。

4. 如果有保存毫秒类似的需求，最好是用时间类型自己的特性，不要直接用字符类型来代替。MySQL 内部的类型转换对资源额外的消耗也是需要考虑的。

# 枚举

枚举类型，也即 enum。适合提前规划好了所有已经知道的值，且未来最好不要加新值的情形。枚举类型有以下特性：

1. 最大占用 2 Byte。

2. 最大支持 65535 个不同元素。

3. MySQL 后台存储以下标的方式，也就是 tinyint 或者 smallint 的方式，下标从 1 开始。

4. 排序时按照下标排序，而不是按照里面元素的数据类型。所以这点要格外注意。

```
mysql> create table t1(
    -> c1 enum('mysql','oracle','dble','postgresql','mongodb','redis','db2','sql server')
    -> );
Query OK, 0 rows affected (0.02 sec)

mysql>
mysql> insert into t1 values (1);
Query OK, 1 row affected (0.01 sec)

mysql>
mysql> insert into t1 values (2);
Query OK, 1 row affected (0.00 sec)

mysql>
mysql> insert into t1 values ('mongodb');
Query OK, 1 row affected (0.01 sec)

mysql>
mysql> insert into t1 values ('db2');
Query OK, 1 row affected (0.01 sec)

mysql> select * from t1 order by c1;
+---------+
| c1      |
+---------+
| mysql   |
| oracle  |
| mongodb |
| db2     |
+---------+
4 rows in set (0.00 sec)

```



# 参考资料

