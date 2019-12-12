---
layout: post
title: MySQL排序-order by原理(part2)
date: 2019-12-12
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
- 通过对返回数据进行排序，也就是通常所说的filesort排序，**所有不是通过索引直接返回排序结果的排序都叫filesort排序**。 filesort并不代表通过磁盘文件进行排序，而只是进行了一个排序操作，至于排序操作是否使用了磁盘文件或者临时表等，则取决于MySQL服务器对排序参数的设置和需要排序数据的大小。

**sort buffer**：MySQL在需要做排序操作时会为每个线程分配一块内存进行排序，叫做`sort buffer`，可以通过配置参数`sort_buffer_size` 设置buffer的大小

```
mysql> show variables like '%sort_buffer%';
+-------------------------+---------+
| Variable_name           | Value   |
+-------------------------+---------+
| innodb_sort_buffer_size | 1048576 |
| myisam_sort_buffer_size | 262144  |
| sort_buffer_size        | 720896  |
+-------------------------+---------+
3 rows in set (0.04 sec)
```

如果要排序的数据量小于 `sort_buffer_size`，排序就在内存中完成。但如果排序数据量太大，内存放不下，则不得不利用磁盘临时文件辅助排序。

> 下面的内容都是从极客时间的课程里拷贝的，比别的文章讲的更简单明了

```
CREATE TABLE `t` (
`id` INT ( 11 ) NOT NULL,
`city` VARCHAR ( 16 ) NOT NULL,
`name` VARCHAR ( 16 ) NOT NULL,
`age` INT ( 11 ) NOT NULL,
`addr` VARCHAR ( 128 ) DEFAULT NULL,
PRIMARY KEY ( `id` ),
KEY `city` ( `city` ) 
) ENGINE = INNODB;
```

对于下面的查询

```
select city,name,age from t where city='杭州' order by name limit 1000;
```

通常情况下，这个语句执行流程如下所示 ：

- 初始化 sort_buffer，确定放入 name、city、age 这三个字段；
- 从索引 city 找到第一个满足 city='杭州’条件的主键 id
- 到主键 id 索引取出整行，取 name、city、age 三个字段的值，存入 sort_buffer 中；
- 从索引 city 取下一个记录的主键 id；
- 重复步骤 3、4 直到 city 的值不满足查询条件为止；
- 对`sort_buffer `中的数据按照字段 name 做快速排序；
- 按照排序结果取前 1000 行返回给客户端。

我们暂且把这个排序过程，称为全字段排序，执行流程的示意图如下所示

![](/assets/images/posts/mysql-sort/mysql-sort-1.jpg)

图中“按 name 排序”这个动作，可能在内存中完成，也可能需要使用外部排序，这取决于排序所需的内存和参数 sort_buffer_size。

我们可以用下面介绍的方法，来确定一个排序语句是否使用了临时文件。

```

/* 打开optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* @a保存Innodb_rows_read的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 执行语句 */
select city, name,age from t where city='杭州' order by name limit 1000; 

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G

/* @b保存Innodb_rows_read的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 计算Innodb_rows_read差值 */
select @b-@a;
```

这个方法是通过查看 OPTIMIZER_TRACE 的结果来确认的，你可以从 number_of_tmp_files 中看到是否使用了临时文件。

![](/assets/images/posts/mysql-sort/mysql-sort-2.png)

number_of_tmp_files 表示的是，排序过程中使用的临时文件数。你一定奇怪，为什么需要 12 个文件？内存放不下时，就需要使用外部排序，外部排序一般使用归并排序算法。可以这么简单理解，MySQL 将需要排序的数据分成 12 份，每一份单独排序后存在这些临时文件中。然后把这 12 个有序文件再合并成一个有序的大文件。
如果 sort_buffer_size 超过了需要排序的数据量的大小，number_of_tmp_files 就是 0，表示排序可以直接在内存中完成。否则就需要放在临时文件中排序。sort_buffer_size 越小，需要分成的份数越多，number_of_tmp_files 的值就越大。

我们的示例表中有 4000 条满足 city='杭州’的记录，所以你可以看到 examined_rows=4000，表示参与排序的行数是 4000 行。

sort_mode 里面的 packed_additional_fields 的意思是，排序过程对字符串做了“紧凑”处理。即使 name 字段的定义是 varchar(16)，在排序过程中还是要按照实际长度来分配空间的。同时，最后一个查询语句 select @b-@a 的返回结果是 4000，表示整个执行过程只扫描了 4000 行。

> 这里需要注意的是，为了避免对结论造成干扰， internal_tmp_disk_storage_engine 设置成 MyISAM。否则，select @b-@a 的结果会显示为 4001。这是因为查询 OPTIMIZER_TRACE 这个表时，需要用到临时表，而 internal_tmp_disk_storage_engine 的默认值是 InnoDB。如果使用的是 InnoDB 引擎的话，把数据从临时表取出来的时候，会让 Innodb_rows_read 的值加 1。

## rowid
排序在上面这个算法过程里面，只对原表的数据读了一遍，剩下的操作都是在 sort_buffer 和临时文件中执行的。但这个算法有一个问题，就是如果查询要返回的字段很多的话，那么 sort_buffer 里面要放的字段数太多，这样内存里能够同时放下的行数很少，要分成很多个临时文件，排序的性能会很差。

所以如果单行很大，这个方法效率不够好。那么，如果 MySQL 认为排序的单行长度太大会怎么做呢？

`max_length_for_sort_data`：MySQL 中专门控制用于排序的行数据的长度的一个参数。它的意思是，如果单行的长度超过这个值，MySQL 就认为单行太大，要换一个算法。

```
mysql> SHOW VARIABLES LIKE '%max_length_for_sort_data%';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| max_length_for_sort_data | 4096  |
+--------------------------+-------+
1 row in set (0.05 sec)
```

city、name、age 这三个字段的定义总长度是 36，我把 max_length_for_sort_data 设置为 16，我们再来看看计算过程有什么改变。

```
SET max_length_for_sort_data = 16;
```

新的算法放入 sort_buffer 的字段，只有要排序的列（即 name 字段）和主键 id。但这时，排序的结果就因为少了 city 和 age 字段的值，不能直接返回了，整个执行流程就变成如下所示的样子：

1. 初始化 sort_buffer，确定放入两个字段，即 name 和 id；
2. 从索引 city 找到第一个满足 city='杭州’条件的主键id；
3. 到主键 id 索引取出整行，取 name、id 这两个字段，存入 sort_buffer 中；
4. 从索引 city 取下一个记录的主键 id；
5. 重复步骤 3、4 直到不满足 city='杭州’条件为止；
6. 对 sort_buffer 中的数据按照字段 name 进行排序；
7. 遍历排序结果，取前 1000 行，并按照 id 的值回到原表中取出 city、name 和 age 三个字段返回给客户端。

![](/assets/images/posts/mysql-sort/mysql-sort-3.jpg)

对比上面全字段排序流程图你会发现，rowid 排序多访问了一次表 t 的主键索引，就是步骤 7。

需要说明的是，最后的“结果集”是一个逻辑概念，实际上 MySQL 服务端从排序后的 sort_buffer 中依次取出 id，然后到原表查到 city、name 和 age 这三个字段的结果，不需要在服务端再耗费内存存储结果，是直接返回给客户端的。

这个时候执行 `select @b-@a`， `examined_rows` 的值还是 4000，表示用于排序的数据是 4000 行。但是 `select @b-@a` 这个语句的值变成 5000 了。因为这时候除了排序过程外，在排序完成后，还要根据 id 去原表取值。由于语句是 limit 1000，因此会多读 1000 行。

![](/assets/images/posts/mysql-sort/mysql-sort-4.png)

从 OPTIMIZER_TRACE 的结果中，你还能看到另外两个信息也变了。

- `sort_mode` 变成了` <sort_key, rowid>`，表示参与排序的只有 name 和 id 这两个字段。
- `number_of_tmp_files` 变成 10 了，是因为这时候参与排序的行数虽然仍然是 4000 行，但是每一行都变小了，因此需要排序的总数据量就变小了，需要的临时文件也相应地变少了。

## 全字段排序 VS rowid 排序

如果 MySQL 实在是担心排序内存太小，会影响排序效率，才会采用 rowid 排序算法，这样排序过程中一次可以排序更多行，但是需要再回到原表去取数据。如果 MySQL 认为内存足够大，会优先选择全字段排序，把需要的字段都放到 sort buffer 中，这样排序后就会直接从内存里面返回查询结果了，不用再回到原表去取数据。这也就体现了 MySQL 的一个设计思想：**如果内存够，就要多利用内存，尽量减少磁盘访问**。对于 InnoDB 表来说，rowid 排序会要求回表多造成磁盘读，因此不会被优先选择。

**如果 sort buffer 无法存放所有数据，那么，当 sort buffer 被填满时，需要对 sort buffer 中的元组进行快速排序，并将排序结果写到一个单独临时文件中，同时保存该文件指针，最后对文件进行归并排序。**

对于全字段排序来讲，其元组数据的长度会比 rowid排序更长，同时 sort buffer 能存放的元组数据更少，意味着产生的临时文件更多.因此，额外的磁盘 I/O 有可能会造成 全字段排序变得更慢，而不是更快。

其实，并不是所有的 order by 语句，都需要排序操作的。从上面分析的执行过程，我们可以看到，MySQL 之所以需要生成临时表，并且在临时表上做排序操作，其原因是原来的数据都是无序的。

```
alter table t add index city_user(city, name);
```

这样整个查询过程的流程就变成了：

1. 从索引 (city,name) 找到第一个满足 city='杭州’条件的主键 id；
2. 到主键 id 索引取出整行，取 name、city、age 三个字段的值，作为结果集的一部分直接返回；
3. 从索引 (city,name) 取下一个记录主键 id；
4. 重复步骤 2、3，直到查到第 1000 条记录，或者是不满足 city='杭州’条件时循环结束。

![](/assets/images/posts/mysql-sort/mysql-sort-5.jpg)

# 本地测试
准备数据
```
DELIMITER ;;
CREATE PROCEDURE idata()
BEGIN
    DECLARE i INT;
    SET i=0;
    WHILE i<4000 DO
        INSERT INTO t VALUES (i,'杭州',concat('edgar615',i),'20','XXX');
        SET i=i+1;
    END WHILE;
END;;
DELIMITER ;

CALL idata();
```

执行分析语句

```
"filesort_summary": {
  "memory_available": 32768,
  "key_size": 512,
  "row_size": 652,
  "max_rows_per_buffer": 50,
  "num_rows_estimate": 16912,
  "num_rows_found": 4000,
  "num_initial_chunks_spilled_to_disk": 68,
  "peak_memory_used": 32888,
  "sort_algorithm": "std::sort",
  "sort_mode": "<fixed_sort_key, packed_additional_fields>"
}
```

1. `num_initial_chunks_spilled_to_disk=60`，说明采用了**外部排序**，使用了**磁盘临时文件**
2. `peak_memory_used > memory_available`：**sort buffer空间不足**
3. 如果`sort_buffer_size`越小，`num_initial_chunks_spilled_to_disk`的值就越大
4. 如果`sort_buffer_size`足够大，那么`num_initial_chunks_spilled_to_disk=0`，采用**内部排序**
5. `num_examined_rows=4000`：**参与排序的行数**
6. `sort_mode`含有的`packed_additional_fields`说明排序过程中对字符串做了紧凑处理： 字段name为`VARCHAR(16)`，在排序过程中还是按照**实际长度**来分配空间

`select @b-@a;`返回4001

`packed_additional_fields`对于额外字段数据类型为：CHAR、VARCHAR、以及可为 NULL 的固定长度数据类型，其字段值长度是可以被压缩的。例如，在无压缩的情况下，字段数据类型为 VARCHAR(255)，当字段值只有 3 个字符时，却需要占用 sort buffer 255 个字符长度的内存空间大小；而在有压缩的情况下，字段值为 3 个字符，却只占用 3 个字符长度 + 2 个字节长度标记的内存空间大小；当字段值为 NULL 时，只需通过一个位掩码来标识。

对于可压缩字段来讲，其真实数据长度会比最大长度更小，这就意味着 sort buffer 可以存放更多元组，临时文件数量更少，

修改`max_length_for_sort_data`

```
SET max_length_for_sort_data=16;
```

```
"filesort_summary": {
  "memory_available": 32768,
  "key_size": 516,
  "row_size": 516,
  "max_rows_per_buffer": 63,
  "num_rows_estimate": 16912,
  "num_rows_found": 4000,
  "num_initial_chunks_spilled_to_disk": 65,
  "peak_memory_used": 32784,
  "sort_algorithm": "std::sort",
  "unpacked_addon_fields": "max_length_for_sort_data",
  "sort_mode": "<fixed_sort_key, rowid>"
}
```

- `"num_initial_chunks_spilled_to_disk": 65` 临时文件变少了
- "sort_mode": "<fixed_sort_key, rowid>" 采用rowid排序
`select @b-@a;`返回5001

关于 filesort 部分的跟踪日志的说明：

```
{
  "join_execution": {
    "select#": 1,
    "steps": [
      {
        "filesort_information": [
          {
            "direction": "desc",         // 排序方向
            "table": "`film`",           // 排序表名
            "field": "release_year"      // 排序字段
          }
        ],
        "filesort_priority_queue_optimization": {  // 文件排序优先级队列优化
          "usable": false,                         // 不可用
          "cause": "not applicable (no LIMIT)"     // 原因是没有使用 LIMIT 关键字
        },
        "filesort_execution": [
        ],
        "filesort_summary": {
          "rows": 10,                              // 排序记录数
          "examined_rows": 10,                     // 检查记录数
          "number_of_tmp_files": 0,                // 临时文件数量
          "sort_buffer_size": 32760,               // 排序缓冲大小
          "sort_mode": "<sort_key, rowid>"         // 排序模式
        }
      }
    ]
  }
}
```


# 参考资料

https://my.oschina.net/lvhuizhenblog/blog/552730

https://zhuanlan.zhihu.com/p/24410681

http://database.51cto.com/art/201710/555357.htm

https://www.36nu.com/post/197.html

https://cloud.tencent.com/developer/article/1072184

https://sq.163yun.com/blog/article/187292293774721024

https://www.zhb127.com/archives/mysql-order-by-filesort-internal.html

https://time.geekbang.org/column/article/73479

http://zhongmingmao.me/2019/02/09/mysql-order-by/

```

```