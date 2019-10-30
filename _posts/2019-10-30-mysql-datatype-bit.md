---
layout: post
title: title: MySQL数据类型-bit（part3）
date: 2019-10-30
categories:
    - MySQL
comments: true
permalink: mysql-datatype-bit.html
---

最近有个业务想到可以用bit类型来存储，以前只知道MySQL有bit，却没用过，这里简单记录一下

MySQL提供了允许您存储位值的BIT类型。BIT(m)可以存储多达m位的值，m的范围在1到64之间。

建了一个测试表
```
mysql> desc bit_test;
+--------------+---------+------+-----+---------+----------------+
| Field        | Type    | Null | Key | Default | Extra          |
+--------------+---------+------+-----+---------+----------------+
| id           | int(11) | NO   | PRI | NULL    | auto_increment |
| province_bit | bit(50) | YES  |     | NULL    |                |
+--------------+---------+------+-----+---------+----------------+
2 rows in set (0.02 sec)
```

# 插入
插入有两种方式，按字符串和按整数插入，如果将值插入到长度小于m位的BIT(m)列中，MySQL将在位值的左侧使用零填充。

```
-- 按二进制
mysql> insert into bit_test(province_bit) values(b'11110001111');
Query OK, 1 row affected (0.03 sec)

-- 按整数
mysql> insert into bit_test(province_bit) values(128);
Query OK, 1 row affected (0.04 sec)

mysql> insert into bit_test(province_bit) values(1551768761);
Query OK, 1 row affected (0.03 sec)
```

注意：`128`和`'128'`是不一样的，我们可以在后面的查询里看到，字符会被转换为acii码
```
mysql> insert into bit_test(province_bit) values('128');
Query OK, 1 row affected (0.03 sec)

mysql> insert into bit_test(province_bit) values('1551768761');
1406 - Data too long for column 'province_bit' at row 1

mysql> insert into bit_test(province_bit) values('a');
Query OK, 1 row affected (0.04 sec)
```

boolean会被转换为0,1

```
mysql> insert into bit_test(province_bit) values(true);
Query OK, 1 row affected (0.03 sec)

mysql> insert into bit_test(province_bit) values(false);
Query OK, 1 row affected (0.03 sec)
```

# 查询

```
-- 默认显示
mysql> select province_bit from bit_test;
+----------------------------------------------------+
| province_bit                                       |
+----------------------------------------------------+
| 00000000000000000000000000000000000000011110001111 |
| 00000000000000000000000000000000000000000010000000 |
| 00000000000000000001011100011111100001110010111001 |
| 00000000000000000000000000001100010011001000111000 |
| 00000000000000000000000000000000000000000001100001 |
| 00000000000000000000000000000000000000000000000001 |
| 00000000000000000000000000000000000000000000000000 |
+----------------------------------------------------+
7 rows in set (0.04 sec)

-- 十进制
mysql> select province_bit + 0 from bit_test;
+------------------+
| province_bit + 0 |
+------------------+
|             1935 |
|              128 |
|       1551768761 |
|          3224120 |
|               97 |
|                1 |
|                0 |
+------------------+
7 rows in set (0.03 sec)

-- 二进制
mysql> select bin(province_bit) from bit_test;
+---------------------------------+
| bin(province_bit)               |
+---------------------------------+
| 11110001111                     |
| 10000000                        |
| 1011100011111100001110010111001 |
| 1100010011001000111000          |
| 1100001                         |
| 1                               |
| 0                               |
+---------------------------------+
7 rows in set (0.22 sec)

-- 八进制
mysql> select oct(province_bit) from bit_test;
+-------------------+
| oct(province_bit) |
+-------------------+
| 3617              |
| 200               |
| 13437416271       |
| 14231070          |
| 141               |
| 1                 |
| 0                 |
+-------------------+
7 rows in set (0.03 sec)

mysql> select hex(province_bit) from bit_test;
+-------------------+
| hex(province_bit) |
+-------------------+
| 78F               |
| 80                |
| 5C7E1CB9          |
| 313238            |
| 61                |
| 1                 |
| 0                 |
+-------------------+
7 rows in set (0.03 sec)
```

```
mysql> select bit_count(province_bit) from bit_test;
+-------------------------+
| bit_count(province_bit) |
+-------------------------+
|                       8 |
|                       1 |
|                      18 |
|                       9 |
|                       3 |
|                       1 |
|                       0 |
+-------------------------+
7 rows in set (0.05 sec)
```

```
mysql> select province_bit & 8 from bit_test;
+------------------+
| province_bit & 8 |
+------------------+
|                8 |
|                0 |
|                8 |
|                8 |
|                0 |
|                0 |
|                0 |
+------------------+
7 rows in set (0.03 sec)

mysql> select * from bit_test where province_bit & 8 = 8;
+----+----------------------------------------------------+
| id | province_bit                                       |
+----+----------------------------------------------------+
|  1 | 00000000000000000000000000000000000000011110001111 |
|  3 | 00000000000000000001011100011111100001110010111001 |
|  4 | 00000000000000000000000000001100010011001000111000 |
+----+----------------------------------------------------+
3 rows in set (0.04 sec)
```