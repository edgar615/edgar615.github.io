---
layout: post
title: MySQL索引-自适应哈希索引(part8)
date: 2019-09-24
categories:
    - MySQL
comments: true
permalink:mysql-index-adaptive-hash-index.html
---

# 哈希索引

哈希表是数组+链表的形式。通过哈希函数计算每个节点数据中键所对应的哈希桶位置，如果出现哈希冲突，就用拉链法来解决

哈希索引是基于哈希表实现，只有精确匹配索引所有列的查询才有效，对于每一行数据，存储引擎都会对所有的索引列计算一个哈希码（hash code)，哈希码是一个较小的值，大部分情况下不同的键值的行计算出来的哈希码是不同的，但是也会有例外，就是说不同列值计算出来的hash值一样的（即所谓的hash冲突），哈希索引将所有的哈希码存储在索引中，同时在哈希表中保存指向每一个数据行的指针，hash很适合做索引，为某一列或几列建立hash索引，就会利用这一列或几列的值通过一定的算法计算出一个hash值，对应一行或几行数据。 

从网上找的一个图
![](/assets/images/posts/mysql-index/hash-index-1.png)

哈希索引检索时不需要想B+Tree那样从根结点开始查找，而是经过计算直接定位，所以速度很快。

但是也有限制：

- 只支持精确查找，不能用于部分查找和范围查找。无法排序和分组。因为原来有序的键值经过哈希算法很可能打乱。
- 如果哈希冲突很多，查找速度很慢。比如在有大量重复键值的情况下。因为这时需要根据hash值找的存储数据的物理位置，然后依次遍历这些数据，找到需要的数据

Mermory默认的索引是Hash索引。

# 自适应哈希索引
看完前面文章介绍的索引，我们 可以知道innodb使用的索引结构是B+树，但其实它还支持另一种索引：自适应哈希索引

哈希索引查找最优情况下是查找一次，而B+树最优情况下的查找次数根据层数决定，因此为了提高查询效率，InnoDB允许使用自适应哈希索引来提高性能

Innodb存储引擎会监控对表上二级索引的查找，如果发现某二级索引被频繁访问，二级索引成为热数据，建立哈希索引可以带来速度的提升

# 参考资料

https://www.cnblogs.com/geaozhang/p/7252389.html

https://www.itread01.com/content/1551986696.html

https://jin-yang.github.io/post/mysql-innodb-adaptive-hash-index_init.html

https://zhuanlan.zhihu.com/p/64027532

