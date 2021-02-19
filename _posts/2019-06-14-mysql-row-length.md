---
layout: post
title: 单条记录最大长度的限制
date: 2019-05-13
categories:
    - MySQL
comments: true
permalink: mysql-row-length.html
---



建一个大表，MySQL会报错

```
mysql> CREATE TABLE t1 (
    -> a VARCHAR(10000),
    -> b VARCHAR(10000),
    -> c VARCHAR(10000),
    -> d VARCHAR(10000),
    -> e VARCHAR(10000),
    -> f VARCHAR(10000),
    -> g VARCHAR(6000)
    -> ) ENGINE=InnoDB;
ERROR 1118 (42000): Row size too large. The maximum row size for the used table type, not counting BLOBs, is 65535. This includes storage overhead, check the manual. You have to change some columns to TEXT or BLOBs

mysql> CREATE TABLE t1 (
    -> a VARCHAR(10000),
    -> b VARCHAR(10000),
    -> c VARCHAR(10000),
    -> d VARCHAR(10000),
    -> e VARCHAR(10000),
    -> f VARCHAR(10000),
    -> g VARCHAR(6000)
    -> ) ENGINE=MyISAM;
ERROR 1118 (42000): Row size too large. The maximum row size for the used table type, not counting BLOBs, is 65535. This includes storage overhead, check the manual. You have to change some columns to TEXT or BLOBs
```

无论是 MySQL 还是 Oracle，或者是 SQL Server，其实都有这么两层存在，一个是 **Server 层**，另一个是**存储引擎层**。

其实也很好理解，可以类比一下我们 Windows 操作系统，比如常说的把 D 盘格式化成 NTFS 或者 FAT32，这个 NTFS 或者 FAT32 就可以理解成数据库中的存储引擎。

> 官方文档说明[Limits on Table Column Count and Row Size]( https://dev.mysql.com/doc/refman/5.7/en/column-count-limit.html)

# 1. MySQL Server 的长度限制

> The internal representation of a MySQL table has a maximum row size limit of 65,535 bytes.

MySQL Server 层的限制比较宽，你的一条记录不要超过 65535 个字节即可。

```
mysql> CREATE TABLE t2 (
    -> c1 VARCHAR(32765) NOT NULL,
    -> c2 VARCHAR(32766) NOT NULL
    -> ) ENGINE = InnoDB CHARACTER SET latin1;
Query OK, 0 rows affected (0.04 sec)
```

行长度=32,765 + 2 bytes(记录长度) + 32,766 + 2 bytes=65535

```
mysql> CREATE TABLE t2 (
    -> c1 VARCHAR(65535) NOT NULL
    -> ) ENGINE = InnoDB CHARACTER SET latin1;
ERROR 1118 (42000): Row size too large. The maximum row size for the used table type, not counting BLOBs, is 65535. This includes storage overhead, check the manual. You have to change some columns to TEXT or BLOBs
```

行长度=65535 + 2 bytes =65537

```
mysql> CREATE TABLE t2 (
    -> c1 VARCHAR(65533) NOT NULL
    -> ) ENGINE = InnoDB CHARACTER SET latin1;
Query OK, 0 rows affected (0.04 sec)
```

**NULL需要额外空间的存储**

```

mysql> CREATE TABLE t2 (
    -> c1 VARCHAR(32765),
    -> c2 VARCHAR(32766)
    -> ) ENGINE = InnoDB CHARACTER SET latin1;
ERROR 1118 (42000): Row size too large. The maximum row size for the used table type, not counting BLOBs, is 65535. This includes storage overhead, check the manual. You have to change some columns to TEXT or BLOBs

mysql> CREATE TABLE t2 (
    -> c1 VARCHAR(32764),
    -> c2 VARCHAR(32766)
    -> ) ENGINE = InnoDB CHARACTER SET latin1;
Query OK, 0 rows affected (0.04 sec)
```



# 2. InnoDB 的长度限制

InnoDB 作为现在官方唯一还在继续开发和支持的存储引擎，其长度限制比较严格，其大致的算法如下

**一条记录的长度，不能超过 innodb_page_size 大小的一半（实际上还要小一点，因为要扣除一些页中元数据信息）**

> 即默认MySQL官方推荐的 16K 的页大小，单条记录的长度不能超过 8126Byte。

```

mysql> CREATE TABLE t4 (
    ->    c1 CHAR(255),c2 CHAR(255),c3 CHAR(255),
    ->    c4 CHAR(255),c5 CHAR(255),c6 CHAR(255),
    ->    c7 CHAR(255),c8 CHAR(255),c9 CHAR(255),
    ->    c10 CHAR(255),c11 CHAR(255),c12 CHAR(255),
    ->    c13 CHAR(255),c14 CHAR(255),c15 CHAR(255),
    ->    c16 CHAR(255),c17 CHAR(255),c18 CHAR(255),
    ->    c19 CHAR(255),c20 CHAR(255),c21 CHAR(255),
    ->    c22 CHAR(255),c23 CHAR(255),c24 CHAR(255),
    ->    c25 CHAR(255),c26 CHAR(255),c27 CHAR(255),
    ->    c28 CHAR(255),c29 CHAR(255),c30 CHAR(255),
    ->    c31 CHAR(255),c32 CHAR(255),c33 CHAR(255)
    -> ) ENGINE=InnoDB DEFAULT CHARSET latin1;
ERROR 1118 (42000): Row size too large (> 8126). Changing some columns to TEXT or BLOB may help. In current row format, BLOB prefix of 0 bytes is stored inline.
```



InnoDB 是**以 B+ 树来组织数据**的，假设允许一行数据填满整个页，那么 InnoDB 中的数据结构将会从 B+树退化为单链表，所以 InnoDB 规定了一个页至少包含两行数据。

那这就好理解了，项目中给出的建表语句的字段中，有好几十个 varhcar(1000) 或者 varchar(2000)，累加起来已经远远超过了 8126 的限制。

# 3. 字段个数的限制

同样，除了长度，对每个表有多少个列的个数也是有限制的，这里简单说一下：

1. **MySQL Server 层**规定一个表的字段个数最大为 4096；

2. **InnoDB 层**规定一个表的字段个数最大为 1017；

> 官方文档相关说明 - (Limits on InnoDB Tables)[https://dev.mysql.com/doc/refman/5.7/en/innodb-restrictions.html]

# 4. TEXT

上面的错误提示说可以将部分字段改为TEXT.

1. 在 COMPACT 格式下，TEXT 字段的前 768 个字节存储在当前记录中，超过的部分存储在**溢出页**(overflow page)中，同时当前页中增加一个 20 个字节的指针（即 SPACEID + PAGEID + OFFSET）和本地长度信息（2 个字节），共计 768 + 20 + 2 = 790 个字节存储在当前记录。

2. 在 DYNAMIC 格式下，一开始会尽可能的存储所有内容，当该记录所在的页快要被填满时，InnoDB 会选择该页中一个最长的字段（所以也有可能是 BLOB 之类的类型），将该字段的所有内容存储到溢出页（overflow page）中，同时在原记录中保留 20 个字节的指针`。

3. 当 TEXT 字段存储的内容不大于 40 个字节时，这 40 个字节都会存储在该记录中，此时该字段的长度为 40 + 1（本地长度信息）= 41 个字节。

> 这里提到一个溢出页的概念，其实就是 MySQL 中的一种数据存储机制，当一条记录中的内容，无法存储在单独的一个页内（比如存储一些大的附件），MySQL 会选择部分列的内容存储到其他数据页中，这种仅保存数据的页就叫溢出页（overflow page）。

# # 4. 参考资料

https://mp.weixin.qq.com/s/w3ij101jzDlbu93i5J7uQg