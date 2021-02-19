---
layout: post
title: InnoDB架构（6） - 文件存储结构
date: 2018-08-06
categories:
    - MySQL
comments: true
permalink: innodb-tablespace.html
---

# 1. 文件存储结构

![](/assets/images/posts/mysql-index/mysql-index-1.jpg)

**表空间**

表空间是Innodb存储引擎逻辑的最高层，所有的数据都存放在表空间中，默认情况下，Innodb存储引擎有一个共享表空间ibdata1,即所有数据都存放在这个表空间中内。如果启用了`innodb_file_per_table`参数，则每张表内的数据可以单独放到一个表空间内（*.ibd）。

- 系统表空间：系统表空间是InnoDB数据字典，双写缓冲区，Change Buffer和undo log的存储区 。属于一种共享表空间。
- 独占表空间：含单个InnoDB表的数据和索引 ，并存储在文件系统中自己的数据文件中。
- 常规表空间：类似于系统表空间，可以存储多个表的数据的一直共享表空间，支持Antelope和Barracuda文件格式。
- undo表空间：undo表空间包含undo log撤销记录的集合，其中包含通过主键索引事务撤销更改的最小信息。

**但请注意，只有数据、索引、和插入缓冲Bitmap放入单独表空间内，其他数据，比如回滚(undo)信息、插入缓冲检索页、系统事物信息，二次写缓冲等还是放在原来的共享表内的。**

**段**

表空间由段组成，常见的段有数据段、索引段、回滚段等。因为InnoDB存储引擎表是索引组织的，因此数据即索引，索引即数据。数据段即为B+树的叶子结点，索引段即为B+树的非叶子结点。

**区**

区是由连续页组成的空间，在任何情况下每个区的大小都为1MB。为了保证区中页的连续性，InnoDB存储引擎一次从磁盘申请4~5个区。默认情况下，InnoDB存储引擎页的大小为16KB，一个区中一共64个连续的区。

**在每个段开始时，先有32个页大小的碎片页（fragment page）来存放数据，当这些页使用完之后才是64个连续页的申请。这样可以避免小表浪费空间**

**页**

页是InnoDB磁盘管理的最小单位。在InnoDB存储引擎中，默认每个页的大小为16KB。可以通过参数`innodb_page_size`将页的大小设置为4K，8K，16K。若设置完成，则所有表中页的大小都固定，不可以对其再次修改。除非通过mysqldump导入和导出操作来产生新的库。

InnoDB存储引擎中，常见的页类型有：数据页，undo页，系统页，事务数据页，插入缓冲位图页，插入缓冲空闲列表页等。

**行**

InnoDB存储引擎是面向列的，数据按行存放，每页中最多存放16KB/2-200行数据

表的行格式决定了其行的物理存储方式，进而会影响查询和DML操作的性能。

**总结一下**

- 一个表的数据页是通过链表连在一起的
- 数据是以行为单位一行一行的存放在磁盘上的块中
- 在访问数据时，一次从磁盘中读出或者写入至少一个完整的页

# 2. 系统表空间

在 MySQL 数据目录下有一个名为 ibdata1 的文件，可以保存一张或者多张表。

```
~# ls -sish /usr/local/mysql/data/ibdata1
2259711 12M /usr/local/mysql/data/ibdata1
```

这个文件就是 MySQL 的系统表空间文件，默认为 1 个。

如果我们不在`my.cnf`文件中指定`innodb_data_home_dir`和`innodb_data_file_path`那么默认会在datadir目录下创建ibdata1 作为innodb tablespace

默认值

```
mysql> show variables like 'innodb_data%';
+-----------------------+------------------------+
| Variable_name         | Value                  |
+-----------------------+------------------------+
| innodb_data_file_path | ibdata1:12M:autoextend |
| innodb_data_home_dir  |                        |
+-----------------------+------------------------+
2 rows in set (0.01 sec)

```

ibdata1可以可以有多个，只需要在配置文件 my.cnf 里面这样定义即可。

```
innodb_data_file_path=ibdata1:200M;ibdata2:200M:autoextend:max:800M
```

指定的文件必须大于10M，如果不受系统文件限制，可以设置大于4G。该变量是mysql服务器容量规划和性能扩展能力的核心要素。通常设置是创建一个数据目录内容的基线大小，在10M到128M之间，第二个文件设置为10M并自动扩展。

```
innodb_data_file_path = ibdata1:128M;ibdata2:10M:autoextend
```

系统表空间不仅可以是文件系统组成的文件，也可以是非文件系统组成的磁盘块，比如裸设备，定义也很简单

```
innodb_data_file_path=/dev/nvme0n1p1:3Gnewraw;/dev/nvme0n1p2:2Gnewraw
```

系统表空间里的具体内容包括：double writer buffer、 change buffer、数据字典（MySQL 8.0 之前）、表数据、表索引。

系统表空间有三个最大的缺点：

- **无法做到自动收缩磁盘空间，造成很大的空间浪费。**

即使它包含的表都被删掉，这部分空间也不会自动释放。

```
~# ls -sish /usr/local/mysql/data/ibdata1
2259711 76M /usr/local/mysql/data/ibdata1
# 删除数据
mysql> drop table user_system;
Query OK, 0 rows affected (0.02 sec)
# 空间未被释放
~# ls -sish /usr/local/mysql/data/ibdata1
2259711 76M /usr/local/mysql/data/ibdata1
```

**如何才能释放 ibdata1 呢?**

这个比较麻烦，而且严重影响服务可用性，大致几个步骤：

1. 用 mysqldump 导出所有表数据；

2. 关闭 MySQL 服务；

3. 设置 ibdata1 为默认大小；

4. source 重新导入数据。

- **扩容时，单表分离速度慢。**

系统表空间在无限制增大导致磁盘满需要扩容时，无法快速的把表从系统表空间里分离出来，必须得经过停服务；改配置；扩容；重新导入数据；启服务等步骤方才可行。

- **多张表的数据写入顺序写。**

对多张表的写入数据依然是顺序写

这就致使 MySQL 发布了单表空间来解决这两个问题。

# 3. 单表空间

单表空间不同于系统表空间，每个表空间和表是一一对应的关系，每张表都有自己的表空间。具体在磁盘上表现为后缀为 .ibd 的文件。

```
~# ls -sish /usr/local/mysql/data/test/*
2259859 80K /usr/local/mysql/data/test/user.ibd
```

单表空间如何应用到具体的表有两种方式

- **方式 1：在配置文件中开启。**

在配置文件中开启单表空间设置参数 `innodb_filer_per_table`，这样默认对当前库下所有表开启单表空间。

```
innodb_file_per_table=1
```

- **方式2：在建表时指定单表空间**

```
CREATE TABLE user (
	id INT(11) NOT NULL AUTO_INCREMENT,
	username VARCHAR(32) NOT NULL,
	age INT(11) NOT NULL,
	PRIMARY KEY (id)
) tablespace innodb_file_per_table;
```

指定系统表空间

```
CREATE TABLE user_system (
	id INT(11) NOT NULL AUTO_INCREMENT,
	username VARCHAR(32) NOT NULL,
	age INT(11) NOT NULL,
	PRIMARY KEY (id)
) tablespace innodb_system;
```

如果在innodb表已创建后设置innodb_file_per_table，那么数据将不会迁移到单独的表空间上，而是续集使用之前的共享表空间。只有新创建的表才会分离到自己的表空间文件。

单表空间除了解决之前说的系统表空间的几个缺点外，还有其他的优点，详细如下：

**`truncate table` 操作比其他的任何表空间都快；**

**可以把不同的表按照使用场景指定在不同的磁盘目录；**

比如日志表放在慢点的磁盘，把需要经常随机读的表放在 SSD 上等。

```
CREATE TABLE sys_log (
	id INT(11) NOT NULL AUTO_INCREMENT,
	content VARCHAR(32) NOT NULL,
	PRIMARY KEY (id)
) data directory = '/root/mysql-files';
```

**可以用 `optimize table` 来收缩或者重建经常增删改查的表。`optimize table user;`**

一般过程是这样的：建立和原来表一样的表结构和数据文件，把真实数据复制到临时文件，再删掉原始表定义和数据文件，最后把临时文件的名字改为和原始表一样的。

```
~# ls -sish /usr/local/mysql/data/test/*
2259859 7.1M /usr/local/mysql/data/test/user.ibd

# 删除所有数据
mysql> delete from user;
Query OK, 21238 rows affected (0.14 sec)

# 表空间未释放
~# ls -sish /usr/local/mysql/data/test/*
2259859 7.1M /usr/local/mysql/data/test/user.ibd

mysql> optimize table user;
+-----------+----------+----------+-------------------------------------------------------------------+
| Table     | Op       | Msg_type | Msg_text                                                          |
+-----------+----------+----------+-------------------------------------------------------------------+
| test.user | optimize | note     | Table does not support optimize, doing recreate + analyze instead |
| test.user | optimize | status   | OK                                                                |
+-----------+----------+----------+-------------------------------------------------------------------+
2 rows in set (0.05 sec)

# 表空间释放
~# ls -sish /usr/local/mysql/data/test/*
2259860 80K /usr/local/mysql/data/test/user.ibd
```

**可以自由移植单表**

并不需要移植整个数据库，可以把单独的表在各个实例之间灵活移植。

```
mysql> create database user2;
Query OK, 1 row affected (0.01 sec)

mysql> use user2;
Database changed

mysql> create table user like test.user;
Query OK, 0 rows affected (0.02 sec)

# 进行表数据移植。
mysql> alter table user discard tablespace;
Query OK, 0 rows affected (0.01 sec)

~# cp -rfp /usr/local/mysql/data/test/user.ibd /usr/local/mysql/data/user2

mysql> alter table user import tablespace;
Query OK, 0 rows affected, 1 warning (0.22 sec)

mysql> select (select count(*) from user2.user) 'user2.user',(select count(*) from test.user) 'test.user';
+------------+-----------+
| user2.user | test.user |
+------------+-----------+
|      29040 |     29040 |
+------------+-----------+
1 row in set (0.06 sec)

```

**单表空间的表可以使用 MySQL 的新特性；**

比如表压缩，大对象更优化的磁盘存储等。

**可以更好的管理和监控单个表的状态；**

比如在 OS 层可以看到表的大小。

**可以解除 InnoDB 系统表空间的大小限制；**

InnoDB 一个表空间最大支持 64TB 数据（针对 16KB 的页大小）。如果是系统表空间，整个实例都被这个限制，单表空间则是针对单个表有 64TB 大小限制。

当然了，单表空间也并不是没有缺点。比如：当多张表被大量的增删改后，表空间会有一定的膨胀；相比系统表空间，打开表需要的文件描述符增多，浪费更多的内存。

# 4. 通用表空间

通用表空间先是出现在 MySQL Cluster 里，也就是 NDB 引擎。从 MySQL 5.7 引入到 InnoDB 引擎。通用表空间和系统表空间一样，也是共享表空间。每个表空间可以包含一张或者多张表，也就是说通用表空间和表之间是一对多的关系。

```
mysql> create tablespace ts1 add datafile '/var/lib/mysql-files/ts1.ibd' engine innodb;
Query OK, 0 rows affected (0.02 sec)

mysql> create table t1(id int,r1 datetime) tablespace ts1;
Query OK, 0 rows affected (0.02 sec)

mysql> create table t2(id int,r1 datetime) tablespace ts1;
Query OK, 0 rows affected (0.03 sec)

mysql> create table t3(id int,r1 datetime) tablespace ts1;
Query OK, 0 rows affected (0.03 sec)
```

通用表空间其实是介于系统表空间和单表空间之间的一种折中的方案。

- 和系统表空间类似，不会自动收缩磁盘空间；
- 和系统表空间类似，可以重命名表空间名字；
- 和单表空间类似，可以很方便把表空间文件定义在 MySQL 数据目录之外；
- 比单表空间占用更少的文件描述符，但是又不能像单表空间那样移植表空间。

幸运的是，可以在这三个表空间里随便切换。不过要注意切换时间点，毕竟切换涉及到数据的迁移，类似 copy 文件对系统的影响。

```
# 表 t1 随时切换各种表空间
mysql> alter table t1 tablespace innodb_file_per_table;
Query OK, 0 rows affected (14.15 sec)
Records: 0 Duplicates: 0 Warnings: 0

mysql> alter table t1 tablespace innodb_system;
Query OK, 0 rows affected (16.95 sec)
Records: 0 Duplicates: 0 Warnings: 0

mysql> alter table t1 tablespace ts1;
Query OK, 0 rows affected (13.98 sec)
Records: 0 Duplicates: 0 Warnings: 0
```

# 5. 表空间销毁

再来看下三种表空间如何销毁：

- 系统表空间无法销毁，除非把里面的内容全部剥离出来；
- 单表空间如果表被删掉了，表空间也就自动销毁；或者是表被移植到其他表空间，单表空间也自动销毁。
- 通用表空间需要引用他的表全部删掉或者移植到其他表空间，才可以被成功删除。

可以通过`innodb_tablespaces `查询表空间的使用情况

# 6. 增加表空间

当没有使用innodb_file_per_table也没有启用自动扩展，那么随着数据的增长，表空间将满了。在这情况下，需要添加额外的表空间来扩展容量。方法如下：

1. 停止mysql服务
2. 备份配置文件，便于出现问题好回退
3. 编辑innodb_data_file_path值，       根据你的环境更改`ibdata1:$size;ibdataN:$size;…ibdataN:$size; `当前定义的表空间或默认表空间是不能改变的，否则启动失败，但是，可以**额外的添加表空间**，ibdataN序列根据当前的数量递增，`$size`自定义。
4. 启动mysql服务
6. 观察mysql错误日志是否有错

例如再添加一个 ibdata2:1G ，如下： 

```
[mysqld]``innodb_data_file_path = ibdata1:12M;ibdata2:1G:autoextend
```


# 参考资料

《MySQL技术内幕 InnoDB存储引擎第二版》

http://blog.codinglabs.org/articles/theory-of-mysql-index.html

https://mp.weixin.qq.com/s/R1zhVWNFtrpCSzM6fPmbBg

https://mp.weixin.qq.com/s/aH87AiBmwCtSf6z9JBc4uQ

https://www.jianshu.com/p/d4cc0ea9d097

https://mp.weixin.qq.com/s/D-pc8yYD9AJVkk8t9_mXzA