---
layout: post
title: MySQL排序-order by limit语句的分页数据重复(part1)
date: 2019-10-29
categories:
    - MySQL
comments: true
permalink: mysql-sort-duplicate.html
---

今天同事像我反应有个业务的分页查询出现了重复数据，在以前的开发中从来没有遇到过，记录一下。

通过日志，找到了SQL（简化了）
```
mysql> select commodity_id, add_on from commodity where add_on > 1555419288 order by add_on limit 1, 5;
+--------------+------------+
| commodity_id | add_on     |
+--------------+------------+
|        31078 | 1555419289 |
|          187 | 1555419290 |
|          194 | 1555419293 |
|        20266 | 1555419293 |
|        19673 | 1555419293 |
+--------------+------------+
5 rows in set (0.04 sec)

mysql> select commodity_id, add_on from commodity where add_on > 1555419288 order by add_on limit 6, 5;
+--------------+------------+
| commodity_id | add_on     |
+--------------+------------+
|        31079 | 1555419293 |
|        20266 | 1555419293 |
|        19673 | 1555419293 |
|        20268 | 1555419294 |
|        31080 | 1555419294 |
+--------------+------------+
5 rows in set (0.05 sec)
```
可以看到19673、20266出现了两次。

搜索了一番，发现是因为MySQL5.6+的版本上**优化器在遇到order by limit语句的时候，做了一个优化，即使用了priority queue**。

> If an index is not used for ORDER BY but a LIMIT clause is also present, the optimizer may be able to avoid using a merge file and sort the rows in memory using an in-memory filesort operation. For details, see The In-Memory filesort Algorithm.[MySQL]

在ORDER BY + LIMIT的查询语句中，如果ORDER BY不能使用索引的话，优化器可能会使用in-memory sort操作，只保留N条数据即可。这样的话只需要 使用sort buffer 少量的内存就可以完成排序。**In memory filesort使用了优先级队列，优先级队列使用了堆排序的排序方法，而堆排序是一个不稳定的排序方法，也就是相同的值可能排序出来的结果和读出来的数据顺序不一致**。

# 解决方法
- 如果在字段添加上索引，就直接按照索引的有序性进行读取并分页，从而可以规避遇到的这个问题
- 在order by指定的排序字段后加一个二级排序字段，保证有序

```
mysql> select commodity_id, add_on from commodity where add_on > 1555419288 order by add_on, commodity_id limit 1, 5;
+--------------+------------+
| commodity_id | add_on     |
+--------------+------------+
|        31078 | 1555419289 |
|          187 | 1555419290 |
|          194 | 1555419293 |
|        19673 | 1555419293 |
|        20266 | 1555419293 |
+--------------+------------+
5 rows in set (0.05 sec)

mysql> select commodity_id, add_on from commodity where add_on > 1555419288 order by add_on, commodity_id limit 6, 5;
+--------------+------------+
| commodity_id | add_on     |
+--------------+------------+
|        31079 | 1555419293 |
|        34556 | 1555419293 |
|        35149 | 1555419293 |
|          196 | 1555419294 |
|        19675 | 1555419294 |
+--------------+------------+
5 rows in set (0.06 sec)
```