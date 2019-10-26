---
layout: post
title: MySQL索引-自适应哈希索引(part9)
date: 2019-10-19
categories:
    - MySQL
comments: true
permalink: mysql-index-adaptive-hash-index.html
---

# 哈希索引

哈希表是数组+链表的形式。通过哈希函数计算每个节点数据中键所对应的哈希桶位置，如果出现哈希冲突，就用拉链法来解决

![](/assets/images/posts/mysql-index/hash-index-1.jpg)

哈希索引是基于哈希表实现，只有精确匹配索引所有列的查询才有效，对于每一行数据，存储引擎都会对所有的索引列计算一个哈希码（hash code)，哈希码是一个较小的值，大部分情况下不同的键值的行计算出来的哈希码是不同的，但是也会有例外，就是说不同列值计算出来的hash值一样的（即所谓的hash冲突），哈希索引将所有的哈希码存储在索引中，同时在哈希表中保存指向每一个数据行的指针，hash很适合做索引，为某一列或几列建立hash索引，就会利用这一列或几列的值通过一定的算法计算出一个hash值，对应一行或几行数据。 

从网上找的一个图
![](/assets/images/posts/mysql-index/hash-index-2.png)

哈希索引检索时不需要想B+Tree那样从根结点开始查找，而是经过计算直接定位，所以速度很快。

但是也有限制：

- 只支持精确查找，不能用于部分查找和范围查找。无法排序和分组。因为原来有序的键值经过哈希算法很可能打乱。
- 如果哈希冲突很多，查找速度很慢。比如在有大量重复键值的情况下。因为这时需要根据hash值找的存储数据的物理位置，然后依次遍历这些数据，找到需要的数据

Mermory默认的索引是Hash索引。

# 自适应哈希索引
看完前面文章介绍的索引，我们 可以知道innodb使用的索引结构是B+树，但其实它还支持另一种索引：自适应哈希索引

哈希索引查找最优情况下是查找一次，而B+树最优情况下的查找次数根据层数决定，因此为了提高查询效率，InnoDB允许使用自适应哈希索引来提高性能

Innodb存储引擎会监控对表上二级索引的查找，如果发现某二级索引被频繁访问，二级索引成为热数据，如果认为建立哈希索引可以提高查询效率，则自动在内存中的 `自适应哈希索引缓冲区` 建立哈希索引。

存储引擎会自动对索引页上的查询进行监控，如果能够通过自适应哈希索引来提高效率，便会自动创建自适应哈希索引，不需要开发人员或者运维人员做任何操作。它是对缓冲池中的 B+ 树页进行创建，且不需要对整个表建立哈希索引，因此它的速度非常快。

可以通过` innodb_adaptive_hash_index` 变量决定是否打开该特性（默认开启），通过 AHI 可以有效提高读取、写入、join 的速度。

```
mysql> show variables like "innodb_adaptive_hash_index";
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| innodb_adaptive_hash_index | ON    |
+----------------------------+-------+
1 row in set (0.05 sec)
```

## 监控
可以通过`SHOW ENGINE INNODB STATUS` 命令查看，在`INSERT BUFFER AND ADAPTIVE HASH INDEX` 段中
```
mysql>  SHOW ENGINE INNODB STATUS;
...
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 48 merges
merged operations:
 insert 53, delete mark 2, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 763673, node heap has 3 buffer(s)
Hash table size 763673, node heap has 114 buffer(s)
Hash table size 763673, node heap has 23 buffer(s)
Hash table size 763673, node heap has 125 buffer(s)
Hash table size 763673, node heap has 8 buffer(s)
Hash table size 763673, node heap has 8 buffer(s)
Hash table size 763673, node heap has 43 buffer(s)
Hash table size 763673, node heap has 68 buffer(s)
10.20 hash searches/s, 11.00 non-hash searches/s
...
```
可以通过 `information_schema.innodb_metrics` 来监控 AHI 模块的运行状态，首先打开监控。
```
----- 打开所有与AHI相关的监控项
mysql> SET GLOBAL innodb_monitor_enable=module_adaptive_hash;
Query OK, 0 rows affected (0.00 sec)

----- 查看监控项是否已经开启
mysql> SELECT name,status FROM information_schema.innodb_metrics WHERE subsystem='adaptive_hash_index';
+------------------------------------------+----------+
| name                                     | status   |
+------------------------------------------+----------+
| adaptive_hash_searches                   | enabled  |
| adaptive_hash_searches_btree             | enabled  |
| adaptive_hash_pages_added                | disabled |
| adaptive_hash_pages_removed              | disabled |
| adaptive_hash_rows_added                 | disabled |
| adaptive_hash_rows_removed               | disabled |
| adaptive_hash_rows_deleted_no_hash_entry | disabled |
| adaptive_hash_rows_updated               | disabled |
+------------------------------------------+----------+
8 rows in set (0.09 sec)

----- 重置所有的计数
mysql> SET GLOBAL innodb_monitor_reset_all = 'adaptive_hash%';
Query OK, 0 rows affected (0.00 sec)
```
该表搜集了 AHI 子系统诸如 AHI 查询次数，更新次数等信息，可以很好的监控其运行状态，在某些负载下，AHI 并不适合打开，关闭 AHI 可以避免额外的维护开销。


# 参考资料

https://www.cnblogs.com/geaozhang/p/7252389.html

https://jin-yang.github.io/post/mysql-innodb-adaptive-hash-index_init.html

https://zhuanlan.zhihu.com/p/64027532

