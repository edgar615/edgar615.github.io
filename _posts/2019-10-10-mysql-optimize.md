---
layout: post
title: MySQL优化
date: 2019-10-10
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
- sort_buffer_size #定义了每个线程排序缓存区的大小，MySQL在有查询、需要做排序操作时才会为每个缓冲区分配内存（直接分配该参数的全部内存）；
- join_buffer_size #定义了每个线程所使用的连接缓冲区的大小，如果一个查询关联了多张表，MySQL会为每张表分配一个连接缓冲，导致一个查询产生了多个连接缓冲；
- read_buffer_size #定义了当对一张MyISAM进行全表扫描时所分配读缓冲池大小，MySQL有查询需要时会为其分配内存，其必须是4k的倍数；
- read_rnd_buffer_size #索引缓冲区大小，MySQL有查询需要时会为其分配内存，只会分配需要的大小。

注意：以上四个参数是为一个线程分配的，如果有100个连接，那么需要×100。

- Innodb_buffer_pool_size #缓冲池大小 

缓冲池的相关资料可以[参考这里](https://edgar615.github.io/innodb-buffer-pool.html)

后续继续补充


# Scheme设计与数据类型优化

如果长度能够满足，整型尽量使用tinyint、smallint、medium_int而非int。注意：对整数类型指定宽度，比如INT(11)，INT(20)，对性能没有任何用处，[参考这里](https://edgar615.github.io/mysql-datatype.html)

# 索引优化

# SQL优化

# 参考资料

https://segmentfault.com/a/1190000013672421

https://mp.weixin.qq.com/s/-rYQQoCqKG4LgOsRziJrtQ

https://mp.weixin.qq.com/s/kKHOoB6WmYXp0crb-jxHSg

https://mp.weixin.qq.com/s/WsQZhZhuzfs2YZgamrGUOw