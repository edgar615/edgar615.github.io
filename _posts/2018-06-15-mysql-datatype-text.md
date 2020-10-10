---
layout: post
title: MySQL数据类型（6）-大对象
date: 2018-06-15
categories:
    - MySQL
comments: true
permalink: mysql-datatype-text.html
---

# 1. TEXT/BLOB 类型

TEXT 和 BLOB 的区别非常简单。TEXT 存储以明文存储，有对应的字符集和校验规则；BLOB 则以二进制存储，没有字符集和排序规则，所有的对比都是以二进制来进行。

创建一张表 c1 字段 f1,f2 分别为 tinytext 和 tinyblob。

```
CREATE TABLE c1 ( f1 TINYTEXT, f2 TINYBLOB );
```

插入数据

```
insert into c1 values ('a','a'),
('b','b'),
('B','B'),
('d','d'),
('F','F'),
('你','你'),
('我','我'),
('是吧','是吧');
```

根据字段 f1 排序

```

mysql> select * from c1 order by f1;
+------+------------+
| f1   | f2         |
+------+------------+
| a    | 0x61       |
| b    | 0x62       |
| B    | 0x42       |
| d    | 0x64       |
| F    | 0x46       |
+------+------------+
5 rows in set (0.00 sec)
```

根据字段 f2 排序

```

mysql> select * from c1 order by f2;
+------+------------+
| f1   | f2         |
+------+------------+
| B    | 0x42       |
| F    | 0x46       |
| a    | 0x61       |
| b    | 0x62       |
| d    | 0x64       |
+------+------------+
5 rows in set (0.00 sec)
```

f1,f2 字段各自排序的结果都不一致。f1 是按照不区分大小写的校验规则，f2 直接二进制检验。

## 1.1. 磁盘空间占用

![](/assets/images/posts/mysql-datatype/mysql-datatype-1.png)

## 1.2. 表的存储格式

- **redundant/compact**

  对 redundant 格式来说，保存大对象的前 768 字节在 InnoDB 数据页，多出来的放在**溢出页**。如果有多个 TEXT/BLOB 字段，那数据页将会变得臃肿不堪，性能影响很大。数据页里几乎全是无用的数据，导致额外的资源消耗。同时如果是主从架构，也会把数据全部同步到从机，对网络也是额外的消耗。所以这种场景下，一般都只是保存大对象的路径到数据库，真实的数据则放在磁盘上。

- **dynamic/compressed**

  对 dynamic 格式来说，如果大对象字段存储数据大小小于 40 字节，那全部放在数据页，剩余的场景，数据页只保留一个 20 字节的指针指向溢出页。 这种场景下，如果每个大对象字段保存的数据小于 40 个字节，也就和 varchar(40)，效果一样。所以用不用大对象不能一概而论。

## 1.3. 索引相关

在大对象字段上建立索引必须是前缀，比如字段 f1 为 text，给前 10 个字符建立索引 idx_f1(f1(10))。

```
mysql> alter table c1 add key idx_f1(f1);
ERROR 1170 (42000): BLOB/TEXT column 'f1' used in key specification without a key length
mysql> alter table c1 add key idx_f1(f1(10));
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

## 1.4. **参数相关**

`mysql_allowed_packet`，这个参数代表 MySQL 服务端和客户端传输的单次数据包上限，如果有 text/blob 字段，此参数设置为最大值 1GB。当然了，必须同时设置客户端和服务端。

# 2. 参考资料

https://mp.weixin.qq.com/s/ycuHpgF5lViEn6JSGCflnw