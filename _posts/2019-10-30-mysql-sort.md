---
layout: post
title: MySQL排序-order by原理(part2)
date: 2019-10-30
categories:
    - MySQL
comments: true
permalink: mysql-sort.html
---

我们先看两段sql
```
mysql> explain select commodity_id from commodity order by seller_id;
+----+-------------+-----------+------------+-------+---------------+------------+---------+------+-------+----------+-------------+
| id | select_type | table     | partitions | type  | possible_keys | key        | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+-----------+------------+-------+---------------+------------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | commodity | NULL       | index | NULL          | idx_seller | 398     | NULL | 22911 |   100.00 | Using index |
+----+-------------+-----------+------------+-------+---------------+------------+---------+------+-------+----------+-------------+
1 row in set (0.05 sec)

mysql> explain select commodity_id from commodity order by add_on;
+----+-------------+-----------+------------+------+---------------+------+---------+------+-------+----------+----------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra          |
+----+-------------+-----------+------------+------+---------------+------+---------+------+-------+----------+----------------+
|  1 | SIMPLE      | commodity | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 22911 |   100.00 | Using filesort |
+----+-------------+-----------+------------+------+---------------+------+---------+------+-------+----------+----------------+
1 row in set (0.05 sec)
```
我们发现第一个sql显示`Using index`，第二个sql显示`Using filesort`。
MySQL 中有两种排序方式：

- 通过有序索引扫描直接返回有序数据，这种方式在使用explain分析查询的时候显示为`using index`， 不需要额外的排序，操作效率较高。
- 通过对返回数据进行排序，也就是通常所说的filesort排序，所有不是通过索引直接返回排序结果的排序都叫filesort排序。 filesort并不代表通过磁盘文件进行排序，而只是进行了一个排序操作，至于排序操作是否使用了磁盘文件或者临时表等，则取决于MySQL服务器对排序参数的设置和需要排序数据的大小。

# 参考资料

https://my.oschina.net/lvhuizhenblog/blog/552730

https://zhuanlan.zhihu.com/p/24410681

http://database.51cto.com/art/201710/555357.htm

https://www.36nu.com/post/197.html

https://cloud.tencent.com/developer/article/1072184

https://sq.163yun.com/blog/article/187292293774721024