---
layout: post
title: MySQL运维与监控（6）- explain format=json
date: 2018-05-18
categories:
    - MySQL
comments: true
permalink: mysql-tps.html
---
```
mysql> explain format=json select * from sbtest3 where id<100 and k<200\G
*************************** 1. row ***************************
EXPLAIN: {
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "26.21"            ##查询总成本
    },
    "table": {
      "table_name": "sbtest3",        ##表名
      "access_type": "range",         ##访问数据的方式是range，即索引范围查找
      "possible_keys": [
        "k_3"
      ],
      "key": "k_3",                   ##使用索引
      "used_key_parts": [
        "k"
      ],
      "key_length": "4",
      "rows_examined_per_scan": 18,   ##扫描 k_3 索引的行数：18（满足特定条件时使用index dive可得到真实行数）
      "rows_produced_per_join": 5,    ##在扫描索引后估算满足id<100条件的行数：5
      "filtered": "33.33",            ##在扫描索引后估算满足其他条件id<100的数据行占比
      "index_condition": "(`sbtest`.`sbtest3`.`k` < 200)",     ##索引条件
      "cost_info": {
        "read_cost": "25.01",         ##这里包含了所有的IO成本+部分CPU成本
        "eval_cost": "1.20",          ##计算扇出的CPU成本
        "prefix_cost": "26.21",       ##read_cost+eval_cost
        "data_read_per_join": "4K"
      },
      "used_columns": [
        "id",
        "k",
        "c",
        "pad"
      ],
      "attached_condition": "(`sbtest`.`sbtest3`.`id` < 100)"
    }
  }
}
```

> MySQL 服务层主要是定义CPU的代价，而MySQL 引擎层主要定义IO代价
>
> MySQL 服务层代价保存在表server_cost中，其具体内容如下：
>
> - row_evaluate_cost (default 0.2) 计算符合条件的行的代价，行数越多，此项代价越大
> - memory_temptable_create_cost (default 2.0) 内存临时表的创建代价
> - memory_temptable_row_cost (default 0.2) 内存临时表的行代价
> - key_compare_cost (default 0.1) 键比较的代价，例如排序
> - disk_temptable_create_cost (default 40.0) 内部myisam或innodb临时表的创建代价
> - disk_temptable_row_cost (default 1.0) 内部myisam或innodb临时表的行代价
>
> MySQL 引擎层代价保存在表engine_cost中，其具体内容如下：
>
> - io_block_read_cost (default 1.0) 从磁盘读数据的代价，对innodb来说，表示从磁盘读一个page的代价
> - memory_block_read_cost (default 1.0) 从内存读数据的代价，对innodb来说，表示从buffer pool读一个page的代价

**eval_cost**

这个很简单，就是计算扇出的 CPU 成本。应用条件 k<200 时，需要扫描索引 18行，这里 18 是精确值（index  dive），然后优化器用了一种叫启发式规则（heuristic）的算法估算出其中满足条件 id<100 的比例为 33.33%，进行 `18*33.33%` 次计算的 CPU 成本等于 `18*33.33%*0.2=1.2`，这里 0.2 是成本常数（即 row_evaluate_cost ）。

**注意：rows_examined_per_scan\*filtered 才是扇出数，不能简单的用 rows_produced_per_join 来表示。**

**read_cost**

这里包含了所有的 IO 成本 +（CPU 成本 - eval_cost）。我们先看下这个SQL的总成本应该怎么算：

访问二级索引 k_3 的成本：

- IO 成本 = `1*1.0`

查询优化器粗暴的认为读取索引的一个范围区间的 I/O 成本和读取一个页面是相同的，这个 SQL 中 k 字段的筛选范围只有 1 个：k < 200，而读取一个页面的 IO 成本为 1.0（即 io_block_read_cost）；

- CPU 成本 = `18*0.2`

从 k 索引中取出 18 行数据后，实际还要再计算一遍，每行计算的成本为 0.2。

然后因为 select * 以及 where id<100 需要的数据都不在索引 k_3 中，所以还需要回表，回表成本：

- IO 成本 = `18*1.0`

从索引中取出满足 k<200 的数据一共是 18 行，所以 = `18*1.0`；

- CPU 成本 = `18*0.2`

从这 18 行完整的数据中计算满足 id<100 的数据，所以也需要计算 18 次。

总成本 = `1*1.0+18*0.2+18*1+18*02=26.2`。因为 eval_cost 算的是扇出的 CPU 成本：`18*33.33%*0.2`，所以 `read_cost = 回表的 CPU 成本 - eval_cost`，也可以这么算 `rows_examined_per_scan*(1-filtered)*0.2`。

**参考资料**

https://mp.weixin.qq.com/s/enVUxwZrkw40XAtV_EeU2A