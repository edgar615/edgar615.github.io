---
layout: post
title:  MySQL DDL为什么成本高？
date: 2018-08-15
categories:
    - MySQL
comments: true
permalink: mysql-ddl-cost.html
---

> 完全复制自https://mp.weixin.qq.com/s/UGkA3zHlp3vIOff9prkxHQ
>
> https://mp.weixin.qq.com/s/i2WOxqsGmvwCHQ4Hik-Qig

众所周知，DDL定义了数据在数据库中的结构、关系以及权限等。比如CREATE，ALTER，DROP等等。
本期我们讨论MySQL 8.0 (使用InnoDB存储引擎)在修改表结构时，究竟会发生什么？

![](/assets/images/posts/mysql-ddl-cost/mysql-ddl-cost-1.png)

既然DDL的作用是改变表结构，那**表结构在InnoDB引擎中是什么样的呢**？如上图，逻辑上，InnoDB表中的数据**可以理解成**按照主键(聚簇索引)顺序存放的，每一行的数据依次排列 (物理上，InnoDB表中的数据按照InnoDB的数据结构B+树进行排列)。

当需要对表增加一列时，会涉及到每一行数据排列的变动，需要重建整张表的数据，可想而知这种变动的成本是高昂的。

然而并不是每一种DDL都要付出这么大的成本，要看具体的分类。

**MySQL 8.0 将DDL用以下五个维度分类讨论**：

- Instant：此变更可以"立刻"完成
- In Place：此变更由InnoDB引擎独立完成，不需要使用Redo log等，可以节省开销
- Rebuild Table：此变更会重建聚簇索引，一般情况下，涉及到数据变更时才需要重建聚簇索引
- Permits Concurrent DML：此变更进行时，是否允许其他DML变更同一张表。此特性关系到变更是否会**长时间**阻塞业务。
- Only Modifies Metadata：此变更是否只变更元信息，不涉及数据变更。

为了容易理解DDL的分类，下图中，我们穷举了MySQL DDL文档中的分类。

列出了**这五个维度的每种组合情况**，每种情况中分别挑选一例典型进行讨论。以下分类是按照DDL的成本从低到高排序。

![](/assets/images/posts/mysql-ddl-cost/mysql-ddl-cost-2.png)

```
ALTER TABLE `t1` ALTER COLUMN `c1` SET DEFAULT '1';
```

修改列的默认值不需要变动已有的数据页，仅需要修改表的元信息即可，所以这是成本最低的一种情况，可以"立刻"完成。

![](/assets/images/posts/mysql-ddl-cost/mysql-ddl-cost-3.png)

```
ALTER TABLE `t1` DROP INDEX `idx1`;
```

删除二级索引除了修改表的元信息之外，需要将对应的二级索引标记为删除状态，因为不需要真的删除，仅仅设置标记量，所以这仍然是一种成本较低的情况。 但由于需要等待所有访问表的事务全部结束后才能成功，所以不算是"立刻"能完成的DDL。

![](/assets/images/posts/mysql-ddl-cost/mysql-ddl-cost-4.png)

```
ALTER TABLE `t1` ADD INDEX `idx1` (`name`(10) ASC) ;
```

创建二级索引除了修改表元信息之外，还需要在存储引擎层建立相应的二级索引结构。

为了支持并发的DML操作，MySQL还需要额外维护一份DDL期间的数据变更日志，在DDL操作最后将并发的DML操作回放至新建的二级索引。不过由于二级索引是通过聚簇索引构造，不需要包含所有的行数据，所以这还不能算是一种较高成本的操作。

![](/assets/images/posts/mysql-ddl-cost/mysql-ddl-cost-5.png)

```
ALTER TABLE `t1`  DROP COLUMN `c1`;
```

删除列和我们之前提到的增加列情况类似，由于需要改动数据行，MySQL在InnoDB引擎内部需要重建聚簇索引 (按照聚簇索引生成临时表,  再取而代之)。同时，为了支持并发的DML操作，还需要维护DDL期间的数据变更日志。可见当数据量较大时，这是一种非常高成本的操作。

![](/assets/images/posts/mysql-ddl-cost/mysql-ddl-cost-6.png)

```
ALTER TABLE `t1` MODIFY COLUMN `c1` INTEGER;
```

变更数据列类型，按照文档描述这是一种无法Inplace的操作，即需要MySQL在server层完成一次表的复制，相比由InnoDB内部完成重建，这种操作需要记录Redo log，占用更多的buffer  pool。不过由于在执行过程中，无法并发DML操作，不需要记录DDL期间的变更日志。即便如此，这仍然是一种高成本的操作。

**运维建议**

- DDL应显式指定ALGORITHM，从低成本(INSTANT)到高成本(COPY)逐一尝试，当不匹配时MySQL会报错。以防我们认为的一个低成本的DDL，因为认为失误而需要重建表，造成运维事故。
- 在以前版本中，MySQL的DDL都需要重建表，所以会建议将一个表的多个变更写在同一句DDL中，用一次重建实施多个变更。 而现在，如果一句DDL中的多个变更的算法不同，那么会使用其中最高成本的算法。 运维中，需要仔细甄别情况，使得一部分变更可以更快完成上线。
- DDL语句允许我们选择锁类型和DDL类型，给予我们更好的自由度。 比如当执行删除列时，MySQL默认使用的是Inplace Rebuild操作，锁级别是None (允许并发读写)。如果业务可以妥协，那么可以将锁级别设置为SHARED (允许并发读但阻塞写)，这样DDL可以更快完成。