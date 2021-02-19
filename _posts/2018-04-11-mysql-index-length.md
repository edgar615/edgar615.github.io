---
layout: post
title: MySQL索引（11）- 索引长度
date: 2018-04-11
categories:
    - MySQL
comments: true
permalink: mysql-index-length.html
---

INNODB的索引会限制单独Key的最大长度为767字节，超过这个长度给出warning，最终索引创建成功，取前缀索引（取前 255 字符）。（MySQL8限制为3072字节）

INNODB会使用编码上限计算，即每个字符占用3字节空间。

所以在utf8编码中，能使用完整索引的最大varchar长度为255（255*3=765）

在utf8mb4编码中，INNODB会使用4字节计算，所以能使用完整索引的最大varchar长度为191（191*4=764）

对于联合索引，总和不得超过 3072 ，否则失败，无法创建。

# 1. MySQL5.7

```
mysql> show variables like '%version%';
+-------------------------+------------------------------+
| Variable_name           | Value                        |
+-------------------------+------------------------------+
| innodb_version          | 5.7.23                       |
| protocol_version        | 10                           |
| slave_type_conversions  |                              |
| tls_version             | TLSv1,TLSv1.1                |
| version                 | 5.7.23-log                   |
| version_comment         | MySQL Community Server (GPL) |
| version_compile_machine | x86_64                       |
| version_compile_os      | Linux                        |
+-------------------------+------------------------------+
8 rows in set (0.06 sec)

mysql> show warnings;
Empty set

mysql> CREATE TABLE `cc` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c` varchar(256) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_c` (`c`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
Query OK, 0 rows affected (0.06 sec)

mysql> show warnings;
Empty set
```

发现没有警告信息，索引也没有变成前缀索引

```
mysql> show index from cc;
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| cc    |          0 | PRIMARY  |            1 | id          | A         |           0 | NULL     | NULL   |      | BTREE      |         |               |
| cc    |          1 | idx_c    |            1 | c           | A         |           0 | NULL     | NULL   | YES  | BTREE      |         |               |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
2 rows in set (0.07 sec)
```

因为MySQL新版本中增加了一个变量`innodb_large_prefix`：如果`innodb_large_prefix`打开，在InnoDB表DYNAMIC或COMPRESSED列格式下，索引前缀最大支持前3072字节；如果不打开的话，在任意列格式下，最多支持前767字节。 这个限制既适用于前缀索引也适用于全列索引。

> Enable this option to allow index key prefixes longer than 767 bytes (up to 3072 bytes) for InnoDB tables that use DYNAMIC or COMPRESSED row format. (Creating such tables also requires the option values innodb_file_format=barracuda and innodb_file_per_table=true.

查看了数据库配置，`innodb_large_prefix` 是开启状态

```
mysql> show variables like 'innodb_large_prefix';
+---------------------+-------+
| Variable_name       | Value |
+---------------------+-------+
| innodb_large_prefix | ON    |
+---------------------+-------+
1 row in set (0.06 sec)
```

关闭后再次测试

```
mysql> set global innodb_large_prefix=OFF;
Query OK, 0 rows affected (0.03 sec)

mysql> show variables like 'innodb_large_prefix';
+---------------------+-------+
| Variable_name       | Value |
+---------------------+-------+
| innodb_large_prefix | OFF   |
+---------------------+-------+
1 row in set (0.06 sec)
```

```
mysql> CREATE TABLE `cc` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c` varchar(256) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_c` (`c`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
1071 - Specified key was too long; max key length is 767 bytes
```

直接报错了和网上的说法不符

在`innodb_large_prefix`开启的情况下，发现单列索引最长只能3072bytes

```
mysql> set global innodb_large_prefix=ON;
Query OK, 0 rows affected (0.03 sec)

mysql> CREATE TABLE `cc2` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c` varchar(1025) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_c` (`c`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
1071 - Specified key was too long; max key length is 3072 bytes
```

测试联合索引

```
mysql> set global innodb_large_prefix=ON;
Query OK, 0 rows affected (0.03 sec)

mysql> CREATE TABLE `test_4` (  
  `a` varchar(255) DEFAULT NULL,  
  `b` varchar(255) DEFAULT NULL,  
  `c` varchar(255) DEFAULT NULL,  
  `d` varchar(300) DEFAULT NULL,  
  KEY `a` (`a`,`b`,`c`,`d`)  
) ENGINE=InnoDB DEFAULT CHARSET=utf8;  
1071 - Specified key was too long; max key length is 3072 bytes

mysql> set global innodb_large_prefix=OFF;
Query OK, 0 rows affected (0.03 sec)

mysql> CREATE TABLE `test_4` (  
  `a` varchar(255) DEFAULT NULL,  
  `b` varchar(255) DEFAULT NULL,  
  `c` varchar(255) DEFAULT NULL,  
  `d` varchar(300) DEFAULT NULL,  
  KEY `a` (`a`,`b`,`c`,`d`)  
) ENGINE=InnoDB DEFAULT CHARSET=utf8;  
1071 - Specified key was too long; max key length is 767 bytes
```

开启`innodb_large_prefix`是联合索引直接报错，关闭`innodb_large_prefix`时单列索引报错

换到阿里云的一台RDS上测试（没有别的5.7机器了）

```
mysql> show variables like '%version%';
+-------------------------+-----------------------+
| Variable_name           | Value                 |
+-------------------------+-----------------------+
| innodb_version          | 5.7.25                |
| protocol_version        | 10                    |
| rds_audit_log_version   | MYSQL_V1              |
| rds_version             | 25                    |
| slave_type_conversions  |                       |
| tls_version             | TLSv1,TLSv1.1,TLSv1.2 |
| version                 | 5.7.25-log            |
| version_comment         | Source distribution   |
| version_compile_machine | x86_64                |
| version_compile_os      | Linux                 |
+-------------------------+-----------------------+
10 rows in set (0.07 sec)

mysql> show variables like 'innodb_large_prefix';
+---------------------+-------+
| Variable_name       | Value |
+---------------------+-------+
| innodb_large_prefix | OFF   |
+---------------------+-------+
1 row in set (0.07 sec)

mysql> CREATE TABLE `cc` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c` varchar(256) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_c` (`c`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
Query OK, 0 rows affected (0.05 sec)

mysql> show warnings;
+---------+------+---------------------------------------------------------+
| Level   | Code | Message                                                 |
+---------+------+---------------------------------------------------------+
| Warning | 1071 | Specified key was too long; max key length is 767 bytes |
+---------+------+---------------------------------------------------------+
1 row in set (0.06 sec)

mysql> show index from cc;
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| cc    |          0 | PRIMARY  |            1 | id          | A         |           0 | NULL     | NULL   |      | BTREE      |         |               |
| cc    |          1 | idx_c    |            1 | c           | A         |           0 |      255 | NULL   | YES  | BTREE      |         |               |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
2 rows in set (0.07 sec)
```

发现创建成功，但是索引只取了255长度的前缀索引

在看联合索引

```
mysql> CREATE TABLE `test_4` (  
  `a` varchar(255) DEFAULT NULL,  
  `b` varchar(255) DEFAULT NULL,  
  `c` varchar(255) DEFAULT NULL,  
  `d` varchar(300) DEFAULT NULL,  
  KEY `a` (`a`,`b`,`c`,`d`)  
) ENGINE=InnoDB DEFAULT CHARSET=utf8;  
Query OK, 0 rows affected (0.04 sec)


mysql> show warnings;
+---------+------+---------------------------------------------------------+
| Level   | Code | Message                                                 |
+---------+------+---------------------------------------------------------+
| Warning | 1071 | Specified key was too long; max key length is 767 bytes |
+---------+------+---------------------------------------------------------+
1 row in set (0.07 sec)

mysql> show index from test_4;
+--------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table  | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+--------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| test_4 |          1 | a        |            1 | a           | A         |           0 | NULL     | NULL   | YES  | BTREE      |         |               |
| test_4 |          1 | a        |            2 | b           | A         |           0 | NULL     | NULL   | YES  | BTREE      |         |               |
| test_4 |          1 | a        |            3 | c           | A         |           0 | NULL     | NULL   | YES  | BTREE      |         |               |
| test_4 |          1 | a        |            4 | d           | A         |           0 |      255 | NULL   | YES  | BTREE      |         |               |
+--------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
4 rows in set (0.07 sec)
```

发现对联合索引的最后一位又做了前缀索引

PS：用root将`innodb_large_prefix`返回没有权限，便没有测试`innodb_large_prefix`打开的情况

在`set global innodb_large_prefix=ON/OFF;`命令后，发现这个参数已经设置为过期了。

```
mysql>  set global innodb_large_prefix=ON;
Query OK, 0 rows affected (0.03 sec)

mysql> show warnings;
+---------+------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Level   | Code | Message                                                                                                                                                         |
+---------+------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Warning |  131 | Using innodb_large_prefix is deprecated and the parameter may be removed in future releases. See http://dev.mysql.com/doc/refman/5.7/en/innodb-file-format.html |
+---------+------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.06 sec)
```

# 2. MySQL8

本地找了台8.0的MySQL

```
mysql> show variables like '%version%';
+--------------------------+------------------------------+
| Variable_name            | Value                        |
+--------------------------+------------------------------+
| immediate_server_version | 999999                       |
| innodb_version           | 8.0.15                       |
| original_server_version  | 999999                       |
| protocol_version         | 10                           |
| slave_type_conversions   |                              |
| tls_version              | TLSv1,TLSv1.1,TLSv1.2        |
| version                  | 8.0.15                       |
| version_comment          | MySQL Community Server - GPL |
| version_compile_machine  | x86_64                       |
| version_compile_os       | Linux                        |
| version_compile_zlib     | 1.2.11                       |
+--------------------------+------------------------------+
11 rows in set (0.04 sec)
```

发现它并没有`innodb_large_prefix`变量

```
mysql> show variables like 'innodb_large_prefix';
Empty set
```

单列索引、联合索引的长度限制为3072

```
mysql> CREATE TABLE `cc` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c` varchar(1025) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_c` (`c`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
1071 - Specified key was too long; max key length is 3072 bytes

mysql> CREATE TABLE `test_4` (  
  `a` varchar(255) DEFAULT NULL,  
  `b` varchar(255) DEFAULT NULL,  
  `c` varchar(255) DEFAULT NULL,  
  `d` varchar(300) DEFAULT NULL,  
  KEY `a` (`a`,`b`,`c`,`d`)  
) ENGINE=InnoDB DEFAULT CHARSET=utf8;  
1071 - Specified key was too long; max key length is 3072 bytes
```

网上搜了一下，说是下面的属性是为了兼容5.1版本的属性，所以在8.0版本直接被移除了

```
innodb_file_format
innodb_file_format_check
innodb_file_format_max
innodb_large_prefix
```

# 3. 为什么

**为什么限制为767**
网上搜了一下，资料较少不知道是否准确，对于单列索引限制为767是因为历史原因，
`767=256x3-1`这个3是字符最大占用空间（utf8）。但是在5.5以后，开始支持4个字节的uutf8。`255x4>767`, 于是增加了一个参数叫做`innodb_large_prefix`

> 256的由来： 只是因为char最大是255，所以以前的程序员以为一个长度为255的index就够用了，所以设置这个256.历史遗留问题。   --- by 阿里-丁奇

**为什么限制为3072**
这个和InnoDB的页大小有关。 我们知道InnoDB一个page的默认大小是16k。**由于是B+树组织，要求叶子节点上一个page至少要包含两条记录**所以一个记录最多不能超过8k。又由于InnoDB的聚簇索引结构，一个二级索引要包含主键索引，因此每个单个索引不能超过 4 k（极端情况，聚集索引和某个二级索引都达到这个限制）。由于需要预留和辅助空间，扣掉后不能超过 3500 ，取个“整数”就是(1024*3)。

由此推算将页大小改为8K，4K的时候索引长度限制也应该相应减小，

> If you reduce the InnoDB page size to 8KB or 4KB by specifying the innodb_page_size option when creating the MySQL instance, the maximum length of the index key is lowered proportionally, based on the limit of 3072 bytes for a 16KB page size. That is, the maximum index key length is 1536 bytes when the page size is 8KB, and 768 bytes when the page size is 4KB.    摘自[官方文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-limits.html)

用docker安装一个页大小为4K的MySQL8，测试一把

```
mysql> show variables like '%version%';
+--------------------------+------------------------------+
| Variable_name            | Value                        |
+--------------------------+------------------------------+
| immediate_server_version | 999999                       |
| innodb_version           | 8.0.18                       |
| original_server_version  | 999999                       |
| protocol_version         | 10                           |
| slave_type_conversions   |                              |
| tls_version              | TLSv1,TLSv1.1,TLSv1.2        |
| version                  | 8.0.18                       |
| version_comment          | MySQL Community Server - GPL |
| version_compile_machine  | x86_64                       |
| version_compile_os       | Linux                        |
| version_compile_zlib     | 1.2.11                       |
+--------------------------+------------------------------+
11 rows in set (0.00 sec)

mysql> show variables like 'innodb_page_size';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| innodb_page_size | 4096  |
+------------------+-------+
1 row in set (0.00 sec)

mysql> CREATE TABLE `cc` (
    ->   `id` int(11) NOT NULL AUTO_INCREMENT,
    ->   `c` varchar(257) DEFAULT NULL,
    ->   PRIMARY KEY (`id`),
    ->   KEY `idx_c` (`c`)
    -> ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
ERROR 1071 (42000): Specified key was too long; max key length is 768 bytes
```
注意：**对于MyISAM引擎，每个列的长度不能大于1000 bytes，所有组成索引列的长度和不能大于1000 bytes，平时基本不用MyISAM引擎，这里就不在测试了**
