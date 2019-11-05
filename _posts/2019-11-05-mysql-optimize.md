---
layout: post
title: MySQL优化
date: 2019-11-05
categories:
    - MySQL
comments: true
permalink: mysql-optimize.html
---

# 影响数据库查询速度的因素

影响数据库查询速度主要有四个因素

1. 服务器硬件
2. 磁盘IO
3. 网卡流量
4. sql查询速度

Cpu负载高，IO负载低可能的原因：

1. 内存不够
2. 磁盘性能差
3. SQL问题--->去数据库层，进一步排查SQL 问题
4. IO出问题了（磁盘到临界了、raid设计不好、raid降级、锁、在单位时间内tps过高）
5. tps过高：大量的小数据IO、大量的全表扫描。

IO负载高，Cpu负载低可能的原因：
1. 大量小的IO写操作：
	- autocommit，产生大量小IO；
	- IO/PS，磁盘的一个定值，硬件出厂的时候，厂家定义的一个每秒最大的IO次数。
2. 大量大的IO 写操作：SQL问题的几率比较大

# 影响MySQL性能的因素

影响MySQL性能的主要有4个因素

1. 服务器硬件（包括系统参数优化）
2. 数据库参数配置
3. 数据库结构设计
4. SQL语句

# 硬件优化

根据数据库类型，主机CPU选择、内存容量选择、磁盘选择：

1. 平衡内存和磁盘资源
2. 随机的I/O和顺序的I/O
3. 主机 RAID卡的BBU（Battery Backup Unit）关闭

## CPU的选择
CPU的两个关键因素：核数、主频

根据不同的业务类型进行选择：

1. CPU密集型：计算比较多，OLTP - 主频很高的cpu、核数还要多
2. IO密集型：查询比较，OLAP - 核数要多，主频不一定高的

## 内存的选择

1. OLAP类型数据库，需要更多内存，和数据获取量级有关
2. OLTP类型数据一般内存是Cpu核心数量的2倍到4倍，没有最佳实践。

## 存储方面

1. 根据存储数据种类的不同，选择不同的存储设备
2. 配置合理的RAID级别（raid5、raid10、热备盘）
3. 对与操作系统来讲，不需要太特殊的选择，最好做好冗余（raid1）（ssd、sas、sata）
4. raid卡：
	- 实现操作系统磁盘的冗余（raid1）
	- 平衡内存和磁盘资源
	-  随机的I/O和顺序的I/O
	-  主机raid卡的BBU（Battery Backup Unit）要关闭。

## 网络设备方面
使用流量支持更高的网络设备（交换机、路由器、网线、网卡、HBA卡）

# 参数配置

## 使用独立表空间

- 系统表空间无法简单的收缩文件大小，造成空间浪费，并会产生大量的磁盘碎片。
- 独立表空间可以通过`optimeze table`收缩系统文件，不需要重启服务器也不会影响对表的正常访问
- 如果对多个表进行刷新时，实际上是顺序进行的，会产生IO瓶颈
- 独立表空间可以同时向多个文件刷新数据。

```
mysql> show variables like 'innodb_file_per_table';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_file_per_table | ON    |
+-----------------------+-------+
1 row in set (0.07 sec)
```

- 如果`innodb_file_per_table` 为 ON 将建立独立的表空间，文件为tablename.ibd
- 如果`innodb_file_per_table` 为 OFF 将数据存储到系统的共享表空间，文件为ibdataX（X为从1开始的整数）

> .frm ：是服务器层面产生的文件，类似服务器层的数据字典，记录表结构。

MySQL5.6后默认使用独立表空间

MySQL没有限制单表最大记录数，它取决于操作系统对文件大小的限制。

![](/assets/images/posts/mysql-optimize/mysql-optimize-3.png)

根据业务情况，数据量达到一定值之后需要考虑分库分表

> 《阿里巴巴Java开发手册》提出单表行数超过500万行或者单表容量超过2GB，才推荐分库分表。

## 连接层
设置合理的并发数和连接方式

```
max_connections           # 最大连接数
max_connect_errors        # 最大错误连接数，能大则大
connect_timeout           # 连接超时
max_user_connections      # 最大用户连接数
skip-name-resolve         # 跳过域名解析
wait_timeout              # 等待超时
back_log                  # 可以在堆栈中的连接数量
```

并发数是指同一时刻数据库能处理多少个请求，由`max_connections`和`max_user_connections`决定。`max_connections`是指MySQL实例的最大连接数，上限值是`16384`，`max_user_connections`是指每个数据库用户的最大连接数。一般要求两者比值超过10%。

**MySQL会为每个连接提供缓冲区，意味着消耗更多的内存。如果连接数设置太高硬件吃不消，太低又不能充分利用硬件。**

```
mysql> show variables like '%max%_connections%';
+----------------------+-------+
| Variable_name        | Value |
+----------------------+-------+
| max_connections      | 1732  |
| max_user_connections | 1732  |
+----------------------+-------+
```

可以在配置文件中修改最大并发数

```
max_connections = 100
max_user_connections = 20
```

## 内存
### key_buffer_size
key_buffer_size 指定索引缓冲区的大小，它决定索引处理的速度，尤其是索引读的速度。
使用查询缓冲，MySQL将查询结果存放在缓冲区中，今后对于同样的SELECT语句（区分大小写），将直接从缓冲区中读取结果。

```
mysql> show variables like 'key_buffer_size';
+-----------------+----------+
| Variable_name   | Value    |
+-----------------+----------+
| key_buffer_size | 16777216 |
+-----------------+----------+
1 row in set (0.03 sec)
```

Key_reads是内存中没有找到索引直接从硬盘读取索引的数量。

```
mysql> show status like 'key_read%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Key_read_requests | 0     |
| Key_reads         | 0     |
+-------------------+-------+
2 rows in set (0.03 sec)
```

查询缓存碎片率= `Qcache_free_blocks/ Qcache_total_blocks* 100%`
查询缓存利用率=` (query_cache_size–Qcache_free_memory) / query_cache_size* 100%`
查询缓存命中率=` (Qcache_hits–Qcache_inserts) / Qcache_hits* 100%`

### sort_buffer_size
sort_buffer_size 定义了每个线程排序缓存区的大小，MySQL在有查询、需要做排序操作时才会为每个缓冲区分配内存（直接分配该参数的全部内存）；增加这个值可以加速ORDER BY或GROUP BY操作。默认数值是2097144(2M)，可改为16777208 (16M)。

### join_buffer_size
join_buffer_size 定义了每个线程所使用的连接缓冲区的大小，如果一个查询关联了多张表，MySQL会为每张表分配一个连接缓冲，导致一个查询产生了多个连接缓冲；

### read_buffer_size 
read_buffer_size定义了当对一张MyISAM进行全表扫描时所分配读缓冲池大小，MySQL有查询需要时会为其分配内存，其必须是4k的倍数；

### read_rnd_buffer_size
read_rnd_buffer_size 索引缓冲区大小，MySQL有查询需要时会为其分配内存，只会分配需要的大小。
record_buffer_size，read_rnd_buffer_size，sort_buffer_size，join_buffer_size为每个线程独占，也就是说，如果有100个线程连接，则占用为`16M*100`。

### table_open_cache
表高速缓存的大小。每当MySQL访问一个表时，如果在表缓冲区中还有空间，该表就被打开并放入其中，这样可以更快地访问表内容。

```
mysql> show global status like 'open%tables%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Open_tables   | 524   |
| Opened_tables | 2679  |
+---------------+-------+
2 rows in set (0.03 sec)

mysql> show variables like 'table_open_cache';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| table_open_cache | 2000  |
+------------------+-------+
1 row in set (0.05 sec)
```

### tmp_table_size
临时表大小。通过设置tmp_table_size选项来增加一张临时表的大小，例如做高级GROUP BY操作生成的临时表。

```
mysql> show global status like 'created_tmp%';
+-------------------------+----------+
| Variable_name           | Value    |
+-------------------------+----------+
| Created_tmp_disk_tables | 13994204 |
| Created_tmp_files       | 1036     |
| Created_tmp_tables      | 42057444 |
+-------------------------+----------+
3 rows in set (0.03 sec)
```

### thread_cache_size
可以复用的保存在缓冲区中的线程的数量。当客户端断开之后，服务器处理此客户的线程将会缓存起来以响应下一个客户而不是销毁（前提是缓存数未达上限）。

```
mysql> show global status like 'Thread%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_cached    | 18    |
| Threads_connected | 33    |
| Threads_created   | 51    |
| Threads_running   | 3     |
+-------------------+-------+
4 rows in set (0.04 sec)

mysql> show variables like 'thread_cache_size';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| thread_cache_size | 100   |
+-------------------+-------+
1 row in set (0.05 sec)
```

### Innodb_buffer_pool_size 
InnoDB使用该参数指定大小的内存来缓冲数据和索引，其对InnoDB的重要性等于key_buffer_size对MyISAM的重要性。

缓冲池的相关资料可以[参考这里](https://edgar615.github.io/innodb-buffer-pool.html)

### Innodb_log_buffer_size

Innodb_log缓存大小，一般为1-8M，默认为1M，对于较大的事务，可以增大缓存大小。可设置为4M或8M。


# Scheme设计与数据类型优化

选择数据类型只要遵循小而简单的原则就好，越小的数据类型通常会更快，占用更少的磁盘、内存，处理时需要的CPU周期也更少。越简单的数据类型在计算时需要更少的CPU周期，比如，整型就比字符操作代价低，因而会使用整型来存储ip地址，使用DATETIME来存储时间，而不是使用字符串。

大多数情况下没有使用枚举类型的必要，其中一个缺点是枚举的字符串列表是固定的，添加和删除字符串（枚举选项）必须使用ALTER TABLE（如果只只是在列表末尾追加元素，不需要重建表）

schema的列不要太多。原因是存储引擎的API工作时需要在服务器层和存储引擎层之间通过行缓冲格式拷贝数据，然后在服务器层将缓冲内容解码成各个列，这个转换过程的代价是非常高的。如果列太多而实际使用的列又很少的话，有可能会导致CPU占用过高

大表ALTER TABLE非常耗时，MySQL执行大部分修改表结果操作的方法是用新的结构创建一个张空表，从旧表中查出所有的数据插入新表，然后再删除旧表。尤其当内存不足而表又很大，而且还有很大索引的情况下，耗时更久，可以采用下面的小技巧：

- 先把数据库拷贝到一台非生产服务器上，在上面做完修改表操作，此时的修改不会影响数据库。修改完毕后再做数据库切换，把非生产数据库切换成生产库。注意：再修改表结构的时候生产库会产生一些新数据，需要通过脚本根据时间区间导入这部分数据
- 影子拷贝：生成一张表结构详图的不同名新数据表，然后导入原表的数据到新表，导入成功后停止数据库，修改原表和新表的名字，将数据访问指向新表，在运行正常后将原表删除。可以通过`online schema change`来完成

## 数值类型

如果长度能够满足，整型尽量使用tinyint、smallint、medium_int而非int。注意：对整数类型指定宽度，比如INT(11)，INT(20)，对性能没有任何用处，[参考这里](https://edgar615.github.io/mysql-datatype.html)

> int(M) 在 integer 数据类型中,M 表示最大显示宽度。在 int(M) 中,M 的值跟 int(M) 所占多少存储空间并无任何关系。和数字位数也无关系 int(3)、int(4)、int(8) 在磁盘上都是占用 4 btyes 的存储空间。

通常来说把可为NULL的列改为NOT NULL不会对性能提升有多少帮助，但MySQL中字段为NULL时依然占用空间，会使索引、索引统计更加复杂。从NULL值更新到非NULL无法做到原地更新，容易发生索引分裂影响性能。因此尽可能将NULL值用有意义的值代替，也能避免SQL语句里面包含`is not null`的判断。

> 如果计划在列上创建索引，就应该将该列设置为NOT NULL

UNSIGNED表示不允许负值，大致可以使正数的上限提高一倍。比如TINYINT存储范围是-128 ~ 127，而UNSIGNED TINYINT存储的范围却是0 - 255

没有太大的必要使用DECIMAL数据类型。即使是在需要存储财务数据时，仍然可以使用BIGINT。比如需要精确到万分之一，那么可以将数据乘以一万然后使用BIGINT存储。这样可以避免浮点数计算不准确和DECIMAL精确计算代价高的问题

- **手机号**：11位手机号如果我们用utf8编码的varchar类型，每位需要3个字节，需要33个字节存放，如果使用bigint只需要8个字节就可以存放
- **IP地址**：同上IP地址也可以用INT（4个字节）来存放，考虑到溢出问题需要用无符号的int
- **年龄、枚举类型**：用tinyint存放，只占用1个字节，无符号的tinyint表述0~255范围，基本够用

## 时间类型
TIMESTAMP使用4个字节存储空间，DATETIME使用8个字节存储空间。因而，TIMESTAMP只能表示1970 - 2038年，比DATETIME表示的范围小得多，而且TIMESTAMP的值因时区不同而不同，TIMESTAMP比DATETIME的空间利用率更高

## 字符串类型
char(N)用来保存固定长度的字符，如果长度不足N的，用空格补齐。varchar(N)用来保存可变长度的字符，它会额外增加1-2个字节来保存字符串的长度。char和varchar占用的字节数根据数据库的编码格式不同而不同，latin1占用1个字节，gbk占用2个字节，utf8占用3个字节

如果字符串长度确定，采用char类型。

如果varchar能够满足，不采用text类型，如果一定要用text类型储存大量数据，表容量会很早涨上去，影响其他字段的查询性能。建议抽取出来放在子表里，用业务主键关联。

text和blob值在执行了大量的删除或更新操作的时候容易影响效率。

删除该类型值会在数据表中留下很大的"空洞"，以后填入这些"空洞"的记录可能长度不同,为了提高性能,建议定期使用 OPTIMIZE TABLE 功能对这类表进行碎片整理。

## 拆分大字段、访问频率低的字段

将大字段、访问频率低的字段拆分到单独的表中存储,分离冷热数据。有利于有效利用缓存，防止读入无用的冷数据，较少磁盘IO，同时保证热数据常驻内存提高缓存命中率。

## 数据文件磁盘分离

MySQL表以数据文件形式存储于文件系统，针对不同的表的读写会打开不同的数据文件。建议对不同的热表进行存储的磁盘分离。通过将不同的热表建立在不同的lun上，分散I/O，这样就能进一步减少I/O消耗的瓶颈。


# 索引优化

分页查询很重要，如果查询数据量超过30%，MYSQL不会使用索引

单表索引数不超过5个、单个索引字段数不超过5个

字符串可使用前缀索引，前缀长度控制在5-8个字符

字段唯一性太低，增加索引没有意义，如：是否删除、性别

避免多个范围条件

合理使用覆盖索引，减少回表次数
> 索引条目远小于数据行大小，如果只读取索引，极大减少数据访问量
> 索引是有按照列值顺序存储的，对于I/O密集型的范围查询要比随机从磁盘读取每一行数据的IO要少的多

使用索引扫描来排序
> MySQL有两种方式可以生产有序的结果集，其一是对结果集进行排序的操作，其二是按照索引顺序扫描得出的结果自然是有序的。如果explain的结果中type列的值为index表示使用了索引扫描来做排序。
>
> 扫描索引本身很快，因为只需要从一条索引记录移动到相邻的下一条记录。但如果索引本身不能覆盖所有需要查询的列，那么就不得不每扫描一条索引记录就回表查询一次对应的行。这个读取操作基本上是随机I/O，因此按照索引顺序读取数据的速度通常要比顺序地全表扫描要慢。
>
> 只有当索引的列顺序和ORDER 
> BY子句的顺序完全一致，并且所有列的排序方向也一样时，才能够使用索引来对结果做排序。如果查询需要关联多张表，则只有ORDER 
> BY子句引用的字段全部为第一张表时，才能使用索引做排序。ORDER 
> BY子句和查询的限制是一样的，都要满足最左前缀的要求（有一种情况例外，就是最左的列被指定为常数，下面是一个简单的示例），其他情况下都需要执行排序操作，而无法利用索引排序。
>
> ```
> // 最左列为常数，索引：(date,staff_id,customer_id)
> select  staff_id,customer_id from demo where date = '2015-06-01' order by staff_id,customer_id
> ```

# SQL优化

## 大批量插入数据

如果同时执行大量的插入，建议使用多个值的INSERT语句(方法二)。这比使用分开INSERT语句快（方法一），一般情况下批量插入效率有几倍的差别。

方法一：

```
insert into tablename values(1,2); 
insert into tablename values(1,3); 
insert into tablename values(1,4);
```

方法二：

```
Insert into tablename values(1,2),(1,3),(1,4); 
```
选择后一种方法的原因：

- 减少SQL语句解析的操作， MySQL没有类似Oracle的share pool，采用方法二，只需要解析一次就能进行数据的插入操作；
- SQL语句较短，可以减少网络传输的IO。

此外，还有以下建议提高插入性能： 

- 通过使用 INSERT DELAYED 语句得到更高的速度。Delayed 的含义是让 insert 语句马上执行，其实数据都被放在内存的队列中，并没有真正写入磁盘；
- 这比每条语句分别插入要快的多，但需要注意，DELAYED关键字只用于MyISAM，MEMORY这类只支持表锁的存储引擎； 
- 将索引文件和数据文件分在不同的磁盘上存放（利用建表中的选项）。

## 避免出现select *

`select *` 操作在任何类型数据库中都不是一个好的SQL开发习惯。使用`select * `取出全部列，会让优化器无法完成索引覆盖扫描这类优化，会影响优化器对执行计划的选择，也会增加网络带宽消耗，更会带来额外的I/O,内存和CPU消耗。建议评估业务实际需要的列数，指定列名以取代`select *`。

> 如果数据表变动，`select *` 可能会导致程序BUG

## 避免使用insert..select..语句

当使用`insert...select...`进行记录的插入时，如果select的表是innodb类型的，不论insert的表是什么类型的表，都会对select的表的纪录进行锁定。

## 适当使用commit

适当使用commit可以释放事务占用的资源而减少消耗，commit后能释放的资源如下：

- 事务占用的undo数据块；
- 事务在redo log中记录的数据块； 
- 释放事务施加的，减少锁争用影响性能。特别是在需要使用delete删除大量数据的时候，必须分解删除量并定期commit。

## 分批处理

不带分页参数的查询或者影响大量数据的update和delete操作，需要拆分处理

业务描述：更新用户所有已过期的优惠券为不可用状态。

SQL语句：

```
update status=0 FROM `coupon` WHERE expire_date <= #{currentDate} and status=1;
```

如果大量优惠券需要更新为不可用状态，执行这条SQL可能会堵死其他SQL，分批处理伪代码如下：
```
int pageNo = 1;
int PAGE_SIZE = 100;
while(true) {
    List batchIdList = queryList('select id FROM `coupon` WHERE expire_date <= #{currentDate} and status = 1 limit #{(pageNo-1) * PAGE_SIZE},#{PAGE_SIZE}');
    if (CollectionUtils.isEmpty(batchIdList)) {
        return;
    }
    update('update status = 0 FROM `coupon` where status = 1 and id in #{batchIdList}')
    pageNo ++;
}
```

## 操作符<>优化
通常<>操作符无法使用索引，举例如下，查询金额不为100元的订单：
```
select id from orders where amount != 100;
```
如果金额为100的订单极少，这种数据分布严重不均的情况下，有可能使用索引。

鉴于这种不确定性，采用union聚合搜索结果，改写方法如下：
```
(select id from orders where amount > 100)
 union all
(select id from orders where amount < 100 and amount > 0)
```

## OR优化

在Innodb引擎下or无法使用组合索引，比如：
```
select id，product_name from orders where mobile_no = '13421800407' or user_id = 100;
```
OR无法命中mobile_no + user_id的组合索引，可采用union，如下所示：
```
(select id，product_name from orders where mobile_no = '13421800407')
 union
(select id，product_name from orders where user_id = 100);
```
此时user_id和mobile_no字段都有索引，查询才最高效。

## union优化

MySQL通过创建并填充临时表的方式来执行union查询。除非确实要消除重复的行，否则建议使用union all。原因在于如果没有all这个关键词，MySQL会给临时表加上distinct选项，这会导致对整个临时表的数据做唯一性校验，这样做的消耗相当高。

## IN优化

IN适合主表大子表小，EXIST适合主表小子表大。由于查询优化器的不断升级，很多场景这两者性能差不多一样了。

尝试改为join查询，举例如下：
```
select id from orders where user_id in (select id from user where level = 'VIP');
```
采用JOIN如下所示：
```
select o.id from orders o left join user u on o.user_id = u.id where u.level = 'VIP';
```

## 不做列运算

通常在查询条件列运算会导致索引失效，如下所示：

查询当日订单
```
select id from order where date_format(create_time，'%Y-%m-%d') = '2019-07-01';
```
date_format函数会导致这个查询无法使用索引，改写后：
```
select id from order where create_time between '2019-07-01 00:00:00' and '2019-07-01 23:59:59';
```
## Like优化

like用于模糊查询，举个例子（field已建立索引）：
```
SELECT column FROM table WHERE field like '%keyword%';
```
这个查询未命中索引，换成下面的写法：
```
SELECT column FROM table WHERE field like 'keyword%';
```
去除了前面的%查询将会命中索引，但是产品经理一定要前后模糊匹配，那只能使用全文索引或者Elasticsearch等搜索引擎

## Join优化

join的实现是采用Nested Loop Join算法，就是通过驱动表的结果集作为基础数据，通过该结数据作为过滤条件到下一个表中循环查询数据，然后合并结果。

如果有多个join，则将前面的结果集作为循环数据，再次到后一个表中查询数据。

驱动表和被驱动表尽可能增加查询条件，满足ON的条件而少用Where，用小结果集驱动大结果集。

被驱动表的join字段上加上索引，无法建立索引的时候，设置足够的Join Buffer Size。

禁止join连接三个以上的表，尝试增加冗余字段。

## Limit优化

limit用于分页查询时越往后翻性能越差，比如：LIMIT 10000 20这样的查询，MySQL需要查询10020条记录然后只返回20条记录，前面的10000条都将被抛弃，这样的代价非常高。解决的原则：缩小扫描范围，如下所示：
```
select * from orders order by id desc limit 100000,10 
```
耗时0.4秒
```
select * from orders order by id desc limit 1000000,10
```
耗时5.2秒

先筛选出ID缩小查询范围，写法如下：
```
select * from orders where id > (select id from orders order by id desc  limit 1000000, 1) order by id desc limit 0,10
```
耗时0.5秒

如果查询条件仅有主键ID，写法如下：
```
select id from orders where id between 1000000 and 1000010 order by id desc
```
耗时0.3秒

## 优化order by

在某些情况中，MySQL 可以使用一个索引来满足 ORDER BY 子句，而不需要额外的排序。where 条件和 order by 使用相同的索引，并且 order by 的顺序和索引顺序相同 ，并且 order by 的字段都是升序或者都是降序。

## count查询
`count(*)`、`count(主键 id) `和` count(1)` 都表示返回满足条件的结果集的总行数；而 `count(字段）`，则表示返回满足条件的数据行里面，参数“字段”不为 NULL 的总个数。

**` count(主键 id)` **：InnoDB 引擎会遍历整张表，把每一行的 id 值都取出来，返回给 server 层。server 层拿到 id 后，判断是不可能为空的，就按行累加。

**` count(1) `**：InnoDB 引擎遍历整张表，但不取值。server 层对于返回的每一行，放一个数字“1”进去，判断是不可能为空的，按行累加。

count(1) 执行得要比 count(主键 id) 快。因为从引擎返回 id 会涉及到解析数据行，以及拷贝字段值的操作。

**` count(字段) `**：

- 如果这个“字段”是定义为 not null 的话，一行行地从记录里面读出这个字段，判断不能为 null，按行累加；
- 如果这个“字段”定义允许为 null，那么执行的时候，判断到有可能是 null，还要把值取出来再判断一下，不是 null 才累加。

**` count(*) `**：并不会把全部字段取出来，而是专门做了优化，不取值。count(*) 肯定不是 null，按行累加。

按照效率排序的话，`count(字段)`&lt;`count(主键 id)`&lt;`count(1)`≈`count(*)`，所以一般尽量使用 count(*)。

## 减少表的锁冲突

对 Innodb 类型的表： 

1. 首先要确认，在对表获取行锁的时候，要尽量的使用索引检索纪录，如果没有使用索引访问，那么即便你只是要更新其中的一行纪录，也是全表锁定的。要确保 sql 是使用索引来访问纪录的，必要的时候，请使用 explain 检查 sql 的执行计划，判断是否按照预期使用了索引。
2. 由于 MySQL 的行锁是针对索引加的锁，不是针对纪录加的锁，所以虽然是访问不同行的纪录，但是如果是相同的索引键，是会被加锁的。
3. 当表有多个索引的时候，不同的事务可以使用不同的索引锁定不同的行，当表有主键或者唯一索引的时候，不是必须使用主键或者唯一索引锁定纪录，其他普通索引同样可以用来检索纪录，并只锁定符合条件的行。 
4. 如果要使用锁定读，（`SELECT ... FOR UPDATE` 或` ... LOCK IN SHARE MODE`），尝试用更低的隔离级别，比如 `READ COMMITTED`。

## 使用truncate代替delete

当删除全表中记录时，使用delete语句的操作会被记录到undo块中，删除记录也记录binlog，当确认需要删除全表时，会产生很大量的binlog并占用大量的undo数据块，此时既没有很好的效率也占用了大量的资源。使用truncate替代，不会记录可恢复的信息，数据不能被恢复。也因此使用truncate操作有其极少的资源占用与极快的时间。另外，使用truncate可以回收表的水位。

# 参考资料

https://segmentfault.com/a/1190000013672421

https://mp.weixin.qq.com/s/-rYQQoCqKG4LgOsRziJrtQ

https://mp.weixin.qq.com/s/kKHOoB6WmYXp0crb-jxHSg

https://mp.weixin.qq.com/s/WsQZhZhuzfs2YZgamrGUOw

https://mp.weixin.qq.com/s/d8csU9_w9gZi6k9SPMajdA