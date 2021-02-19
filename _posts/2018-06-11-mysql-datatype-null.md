---
layout: post
title: MySQL数据类型（2）- NULL
date: 2018-06-11
categories:
    - MySQL
comments: true
permalink: mysql-datatype-null.html
---

# 1. NULL 与空字符存储上的区别

表中如果允许字段为 NULL，会为每行记录分配 NULL 标志位。NULL 除了在每行的行首存有 NULL 标志位，实际存储不占有任何空间。如果表中所有字段都是非 NULL，就不存在这个标示位了。

# 2. **NULL使用上的一些问题。**

数值类型，对一个允许为NULL的字段进行min、max、sum、加减、order by、group by、distinct 等操作的时候。字段值为非 NULL 值时，操作很明确。如果使用 NULL， 需要清楚的知道如下规则：

## 2.1. 数值类型，以 INT 列为例

准备数据

```
CREATE TABLE `t1` (
	`id` int(11) NOT NULL AUTO_INCREMENT,
	`name` varchar(20) DEFAULT NULL,
	`number` int(11) DEFAULT NULL,
	PRIMARY KEY (`id`)
);

insert t1(name, number) values('a', null);
insert t1(name, number) values('b', null);
insert t1(name, number) values('c', 5);
insert t1(name, number) values('d', 1);
```

**1) min / max / sum / avg函数中 NULL 值会被直接忽略掉**

```
mysql> select min(number) from t1;
+-------------+
| min(number) |
+-------------+
|           1 |
+-------------+
1 row in set (0.00 sec)

mysql> select max(number) from t1;
+-------------+
| max(number) |
+-------------+
|           5 |
+-------------+
1 row in set (0.00 sec)

mysql> select sum(number) from t1;
+-------------+
| sum(number) |
+-------------+
|           6 |
+-------------+
1 row in set (0.00 sec)

mysql> select avg(number) from t1;
+-------------+
| avg(number) |
+-------------+
|      3.0000 |
+-------------+
1 row in set (0.00 sec)
```

对于min / max / sum还好理解，但是avg就和我们实际理解有出入了。

**2) 对 NULL 做加减操作,如 1 + NULL，结果仍是 NULL**

```

mysql> select 1 + NULL;
+----------+
| 1 + NULL |
+----------+
|     NULL |
+----------+
1 row in set (0.00 sec)
```

 **3) order by 以升序检索字段的时候 NULL 会排在最前面（倒序相反）**

```
mysql> select * from t1 order by number;
+----+------+--------+
| id | name | number |
+----+------+--------+
|  1 | a    |   NULL |
|  2 | b    |   NULL |
|  4 | d    |      1 |
|  3 | c    |      5 |
+----+------+--------+
4 rows in set (0.00 sec)

mysql> select * from t1 order by number desc;
+----+------+--------+
| id | name | number |
+----+------+--------+
|  3 | c    |      5 |
|  4 | d    |      1 |
|  1 | a    |   NULL |
|  2 | b    |   NULL |
+----+------+--------+
4 rows in set (0.00 sec)
```

**4) group by / distinct 时，NULL 值被视为相同的值**

```
mysql> select distinct number from t1;
+--------+
| number |
+--------+
|   NULL |
|      5 |
|      1 |
+--------+
3 rows in set (0.00 sec)

# count(*)和count(number)有区别，在后面的文章会解释
mysql> select number, count(*) from t1 group by number;
+--------+----------+
| number | count(*) |
+--------+----------+
|   NULL |        2 |
|      5 |        1 |
|      1 |        1 |
+--------+----------+
3 rows in set (0.00 sec)
```

## 2.2. 字符类型

**1) 字段是字符时，你无法一目了然的区分这个值到底是 NULL ，还是字符串 'NULL'**

```
mysql> insert into t1 (name,number) values ('NULL',5);
Query OK, 1 row affected (0.01 sec)

mysql> insert into t1 (number) values (6);
Query OK, 1 row affected (0.01 sec)

mysql> select * from t1 where number in (5,6);
+----+------+--------+
| id | name | number |
+----+------+--------+
|  3 | c    |      5 |
|  5 | NULL |      5 |
|  6 | NULL |      6 |
+----+------+--------+
3 rows in set (0.00 sec)
```

**2) 统计包含 NULL 字段的值，NULL 值不包括在里面**

```
mysql> select count(*) from t1;
+----------+
| count(*) |
+----------+
|        6 |
+----------+
1 row in set (0.01 sec)

mysql> select count(name) from t1;
+-------------+
| count(name) |
+-------------+
|           5 |
+-------------+
1 row in set (0.00 sec)

mysql> select count(number) from t1;
+---------------+
| count(number) |
+---------------+
|             4 |
+---------------+
1 row in set (0.00 sec)

```

**3) 如果你用 length 去统计一个 VARCHAR 的长度时，NULL 返回的将不是数字**

```

mysql> select length(name) from t1 where name is null;
+--------------+
| length(name) |
+--------------+
|         NULL |
+--------------+
1 row in set (0.00 sec)

```

**4) 不等于查询返回不符合预期的结果**

```
mysql> select * from t1 where name <> 'a';
+----+------+--------+
| id | name | number |
+----+------+--------+
|  2 | b    |      2 |
|  3 | c    |      3 |
|  4 | d    |      4 |
+----+------+--------+
3 rows in set (0.00 sec)

mysql> select * from t1;
+----+------+--------+
| id | name | number |
+----+------+--------+
|  1 | a    |      1 |
|  2 | b    |      2 |
|  3 | c    |      3 |
|  4 | d    |      4 |
|  5 | NULL |      5 |
+----+------+--------+
5 rows in set (0.00 sec)

```



# 3. 包含 NULL 的索引列

对包含 NULL 列建立索引，比不包含的 NULL 的字段，要多占用一个 BIT 位来存储

**包含null**

```
CREATE TABLE `t1` (
	`id` int(11) NOT NULL AUTO_INCREMENT,
	`name` varchar(20) DEFAULT NULL,
	`number` int(11) DEFAULT NULL,
	PRIMARY KEY (`id`),
	KEY idx_name(name)
);

insert t1(name, number) values('a', 1);
insert t1(name, number) values('b', 2);
insert t1(name, number) values('c', 3);
insert t1(name, number) values('d', 4);


mysql> pager grep -i 'key_len'
PAGER set to 'grep -i 'key_len''
mysql> explain select * from t1 where name = ''\G
      key_len: 63
1 row in set, 1 warning (0.00 sec)
```

**不包含null**

```
CREATE TABLE `t2` (
	`id` int(11) NOT NULL AUTO_INCREMENT,
	`name` varchar(20) NOT NULL,
	`number` int(11) NOT NULL,
	PRIMARY KEY (`id`),
	KEY idx_name(name)
);

insert t2(name, number) values('a', 1);
insert t2(name, number) values('b', 2);
insert t2(name, number) values('c', 3);
insert t2(name, number) values('d', 4);


mysql> pager grep -i 'key_len'
PAGER set to 'grep -i 'key_len''
mysql> explain select * from t2 where name = ''\G
      key_len: 62
1 row in set, 1 warning (0.00 sec)
```

# 4. 总结

NULL 本身是一个特殊值，MySQL 采用特殊的方法来处理 NULL 值。从理解肉眼判断，操作符运算等操作上，可能和我们预期的效果不一致。可能会给我们项目上的操作不符合预期。

你必须要使用 IS NULL / IS NOT NULL 这种与普通 SQL 大相径庭的方式去处理 NULL。

尽管在存储空间上，在索引性能上可能并不比空值差，但是为了避免其身上特殊性，给项目带来不确定因素，**因此建议默认值不要使用 NULL**。

# 5. 参考资料

https://mp.weixin.qq.com/s/vCAiisq_UgATsisXNi1RRA