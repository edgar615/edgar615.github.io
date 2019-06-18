---
layout: post
title: MySQL分页优化
date: 2019-06-18
categories:
    - MySQL
comments: true
permalink: mysql-pagination-optimize.html
---

对于MySQL分页，我们最常用的方式是

1. 计算总数 `select count(*) from table where ...`
2. 计算得出偏移量`select * from table where ... limit 100,10`

上面的语句在数据量小的时候没有问题，但在数量大的时候会存在明显的性能问题

# LIMIT

首先我们来看`limit语句`

`purchase_order`有100万左右数据，在没有索引的时候

```
mysql> select * from purchase_order limit 100, 10;
10 rows in set (0.08 sec)
mysql> select * from purchase_order limit 10000, 10;
10 rows in set (0.10 sec)
mysql> select * from purchase_order limit 100000, 10;
10 rows in set (0.20 sec)
mysql> select * from purchase_order limit 500000, 10;
10 rows in set (0.78 sec)
mysql> select * from purchase_order limit 990000, 10;
10 rows in set (1.37 sec)
```

可以看到随着偏移量的增加，查询耗时也越长

```
mysql> explain select * from purchase_order limit 990000, 10;
+----+-------------+----------------+------------+------+---------------+------+---------+------+--------+----------+-------+
| id | select_type | table          | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra |
+----+-------------+----------------+------------+------+---------------+------+---------+------+--------+----------+-------+
|  1 | SIMPLE      | purchase_order | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 995038 |   100.00 | NULL  |
+----+-------------+----------------+------------+------+---------------+------+---------+------+--------+----------+-------+
1 row in set (0.07 sec)
```

为什么 offset 偏大之后 limit 查找会变慢？这需要了解 limit 操作是如何运作的，以下面这句查询为例：

```
select * from purchase_order limit 990000, 10;
```

这句 SQL 的执行逻辑是
 1.从数据表中读取第N条数据添加到数据集中
 2.重复第一步直到 N = 990000+ 10
 3.根据 offset 抛弃前面 990000条数
 4.返回剩余的 10 条数据

显然，导致这句 SQL 速度慢的问题出现在第二步！这前面的 990000条数据完全对本次查询没有意义，但是却占据了绝大部分的查询时间！如何解决？首先我们得了解为什么数据库为什么会这样查询。

## 优化方式

我们先看看下面这个语句

```
mysql> select purchase_order_id from purchase_order limit 990000, 10;
10 rows in set (0.34 sec)
```

比`select * from purchase_order limit 990000, 10;`要快了不少，因为加载到数据集中的数据量更小

```
mysql> explain select purchase_order_id from purchase_order limit 990000, 10;
+----+-------------+----------------+------------+-------+---------------+---------+---------+------+--------+----------+-------------+
| id | select_type | table          | partitions | type  | possible_keys | key     | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+----------------+------------+-------+---------------+---------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | purchase_order | NULL       | index | NULL          | PRIMARY | 8       | NULL | 995038 |   100.00 | Using index |
+----+-------------+----------------+------------+-------+---------------+---------+---------+------+--------+----------+-------------+
1 row in set (0.07 sec)
```

所以我们可以采用子查询的方式优化

```
mysql> select * from purchase_order where purchase_order_id in (select purchase_order_id from purchase_order limit 990000, 10);
1235 - This version of MySQL doesn't yet support 'LIMIT & IN/ALL/ANY/SOME subquery'
```

o(╯□╰)o，不支持这种查询好尴尬

换个方式

```
mysql> select * from purchase_order o inner join (select purchase_order_id from purchase_order limit 990000, 10) x on o.purchase_order_id = x.purchase_order_id;
10 rows in set (0.34 sec)

mysql> explain select * from purchase_order o inner join (select purchase_order_id from purchase_order limit 990000, 10) x on o.purchase_order_id = x.purchase_order_id;
+----+-------------+----------------+------------+--------+---------------+---------+---------+---------------------+--------+----------+-------------+
| id | select_type | table          | partitions | type   | possible_keys | key     | key_len | ref                 | rows   | filtered | Extra       |
+----+-------------+----------------+------------+--------+---------------+---------+---------+---------------------+--------+----------+-------------+
|  1 | PRIMARY     | <derived2>     | NULL       | ALL    | NULL          | NULL    | NULL    | NULL                | 990010 |   100.00 | NULL        |
|  1 | PRIMARY     | o              | NULL       | eq_ref | PRIMARY       | PRIMARY | 8       | x.purchase_order_id |      1 |   100.00 | NULL        |
|  2 | DERIVED     | purchase_order | NULL       | index  | NULL          | PRIMARY | 8       | NULL                | 995038 |   100.00 | Using index |
```

如果我们的ID是有序的，我们还可以采用下面的方式优化

```
mysql> select * from purchase_order where purchase_order_id > 990000 limit 10;
10 rows in set (0.08 sec)
```

**我一般采用N+1查询，将N做cache**

# select count(*)

100W的数据效果不明显，我将数据增加到1000W左右

```
mysql> select count(*) from purchase_order;
+----------+
| count(*) |
+----------+
|  9989998 |
+----------+
1 row in set (3.04 sec)
mysql> select count(*) from purchase_order where seller_id = 1;
+----------+
| count(*) |
+----------+
|    10195 |
+----------+
1 row in set (3.24 sec)

mysql> select count(*) from purchase_order where seller_id > 1;
+----------+
| count(*) |
+----------+
|  9969972 |
+----------+
1 row in set (3.45 sec)

mysql> select count(*) from purchase_order where seller_id > 1 and buyer_id < 999;
+----------+
| count(*) |
+----------+
|  9969972 |
+----------+
1 row in set (4.13 sec)
```

可以看到`count(*)`耗时都比较长，因为**`count(*)`是表扫描**

从 MVCC 机制与行可见性问题中可得到原因，每个事务所看到的行可能是不一样的，其 count( * ) 结果也可能是不同的；反过来看，则是 MySQL-Server 端无法在同一时刻对所有用户线程提供一个统一的读视图，也就无法提供一个统一的 count 值。所以InnoDB无法想myisam一样维护一个row_count变量

对于多个访问 MySQL 的用户线程 `COUNT( * )` 而言，决定它们各自的结果的因素有几个:

- 一组事务执行前的数据状态(初始数据状态)。

- 有时间重叠的事务们的执行序列 (操作时序，事务理论表明 并发事务操作的可串行化是正确性的必要条件)。

- 事务们各自的隔离级别(每个操作的输入)。

其中 1、2 对于 Server 而言都是全局或者说可控的，只有 3 是每个用户线程中事务所独有的属性，这是 Server 端不可控的因素，因此 Server 端也就对每个 COUNT( * ) 结果不可控了。

两个`count(*)`的问题

**Q：InnoDB-COUNT( \* ) 属 table scan 操作，是否会将现有 Buffer Pool 中其它用户线程所需热点页从 LRU-list 中挤占掉，从而其它用户线程还需从磁盘 load 一次，突然加重 IO 消耗，可能对现有请求造成阻塞？**

**A：**MySQL 有这样的优化策略，将扫表操作所 load 的 page 放在 LRU-list 的 oung/old 的交界处 ( LRU 尾部约 3/8 处 )。这样用户线程所需的热点页仍然在 LRU-list-young 区域，而扫表操作不断 load 的页则会不断冲刷 old 区域的页，这部分的页本身就是被认为非热点的页，因此也相对符合逻辑。

**Q：InnoDB-COUNT( \* ) 是否会像 SELECT * FROM t 那样读取存储大字段的溢出页(如果存在)？**

**A：否。**因为 InnoDB-COUNT( * ) 只需要数行数，而每一行的主键肯定不是 NULL，因此只需要读主键索引页内的行数据，而无需读取额外的溢出页。

# Yahoo的优化方案

很早的时候就看到过Yahoo的分页优化方案，这次百度了好久才找到原文档。简单抄录一下

- 使用索引过滤
- 使用相同的索引来排序
- 不显示总页数，一般用户不会关心总页数，就好似我们百度，Google根本不会关心搜到了多少条记录（当然有些产品或需求方觉得很重要，泪奔o(╥﹏╥)o）
- 使用`LIMIT N`替换`LIMIT M,N`



## 使用`LIMIT N`替换`LIMIT M,N`

用户每次分页请求时记录最大的ID`last_seen`，那么下一页的请求

```
http://domain.com/forum?page=2&last_seen=100&dir=next
```

对应的sql可以写成

```
WHERE id < 100 /* last_seen *
ORDER BY id DESC LIMIT $page_size /* No OFFSET*/
```

上一页的请求

```
http://domain.com/forum?page=1&last_seen=98&dir=prev
```

对应的sql可以写成

```
*/
Prev Page:
http://domain.com/forum?page=1&last_seen=98&dir=prev
WHERE id > 98 /* last_seen *
ORDER BY id ASC LIMIT $page_size /* No OFFSET* 
```

对于非唯一索引的数据例如`thumbs_up `

```
WHERE thumbs_up < 98
ORDER BY thumbs_up DESC /* It will return few seen rows */
```

可以改为

```
WHERE thumbs_up <= 98
AND <extra_con>
ORDER BY thumbs_up DESC
```

将`thumbs_up`作为major条件，将`id`作为minor条件

第一页的sql

```
SELECT thumbs_up, id
FROM message
ORDER BY thumbs_up DESC, id DESC
LIMIT $page_size
+-----------+----+
| thumbs_up | id |
+-----------+----+
| 99 | 14 |
| 99 | 2 |
| 98 | 18 |
| 98 | 15 |
| 98 | 13 |
+-----------+----+
```

下一页的sql

```
SELECT thumbs_up, id
FROM message
WHERE thumbs_up <= 98 AND (id < 13 OR thumbs_up < 98)
ORDER BY thumbs_up DESC, id DESC
LIMIT $page_size
+-----------+----+
| thumbs_up | id |
+-----------+----+
| 98 | 10 |
| 98 | 6 |
| 97 | 17 |
```

上面的sql可以继续优化

```
SELECT m2.* FROM message m1, message m2
WHERE m1.id = m2.id
AND m1.thumbs_up <= 98
AND (m1.id < 13 OR m1.thumbs_up < 98)
ORDER BY m1.thumbs_up DESC, m1.id DESC
LIMIT 20;
```

