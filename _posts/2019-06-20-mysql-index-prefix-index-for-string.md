---
layout: post
title: MySQL索引-字符串前缀索引（part6）
date: 2019-06-20
categories:
    - MySQL
comments: true
permalink: mysql-index-prefix-index-for-string.html
---

MySQL 前缀索引能有效减小索引文件的大小，提高索引的速度。

但是前缀索引也有它的坏处：MySQL 不能在 ORDER BY 或 GROUP BY 中使用前缀索引，也不能把它们用作覆盖索引(Covering Index)。

语法
```
ALTER TABLE table_name ADD KEY(column_name(prefix_length));
```
在MySQL中，前缀长度最大值为255字节。对于存储引擎为MyISAM或InnoDB的数据表，前缀最长为1000字节。 必须注意的是，在MySQL中，对于TEXT和BLOB这种大数据类型的字段，必须给出前缀长度(length)才能成功创建索引

**如何确定前缀索引长度？**

可以通过计算选择性来确定前缀索引的选择性，计算方法如下

全列选择性：
```
SELECT COUNT(DISTINCT column_name) / COUNT(*) FROM table_name;
```
某一长度前缀的选择性
```
SELECT COUNT(DISTINCT LEFT(column_name, prefix_length)) / COUNT(*) FROM table_name;
```
多个长度前缀的选择性
```
select count(distinct left ([column], 4)) / count(*) as len4 ,   
     count(distinct left ([column], 5)) / count(*) as len5 ,  
     count(distinct left ([column], 6)) / count(*) as len6 ,  
     count(distinct left ([column], 7)) / count(*) as len7 ,  
     count(distinct left ([column], 8)) / count(*) as len8 from [tableName]  
```
当前缀的选择性越接近全列选择性的时候，索引效果越好。

# 案例
> 下面基本全部摘自极客时间的《mysql实战45讲》

系统的用户表支持邮箱登录

```
mysql> create table SUser(
ID bigint unsigned primary key,
email varchar(64), 
... 
)engine=innodb; 

mysql> select f1, f2 from SUser where email='xxx';
```
我们可以建立两种索引
```
mysql> alter table SUser add index index1(email);
```
或
```
mysql> alter table SUser add index index2(email(6));
```
index1 索引里面，包含了每个记录的整个字符串，而index2索引里面对每个记录只取前6个字节，如下图所示

index1

![](/assets/images/posts/mysql-index/string-index-1.png)

index2

![](/assets/images/posts/mysql-index/string-index-2.png)

我们可以看到

- 全字段索引占用空间大，前缀索引占用空间小；
- 全字段索引查询效率高，前缀索引则会增加额外的记录扫描次数。

接下来，我们再看看下面这个语句，在这两个索引定义下分别是怎么执行下面语句的

```
select id,name,email from SUser where email='zhangssxyz@xxx.com';
```

**全字段索引index1**

1. 从 index1 索引树中找到索引值是 zhangssxyz@xxx.com 的记录，然后得到主键值；
2. 根据主键值获取到该行的完整数据（回表），再判断 email 是否满足条件，将这行记录加入结果集；
3. 沿着索引树继续查找下一条满足条件的数据，若不满足，循环结束；

**前缀索引index2**

1. 从 index2 索引树上查找索引值是 zhangs  的记录，找到一条后，得到主键值；
2. 根据主键值获取到该行的完整数据（回表），再判断 email 是否满足条件，将这行记录加入结果集；
3. 沿着索引树继续查找下一条满足条件的数据，发现仍然满足条件，重复上面的操作；
3. 重复上一步，直到在 index2 上取到的值不满足条件，循环结束。
很明显，使用前缀索引，导致查询次数增多。

对于这个查询语句来说，如果你定义的 index2 不是 email(6) 而是 email(7），也就是说取 email 字段的前 7 个字节来构建索引的话，即满足前缀’zhangss’的记录只有一个，也能够直接查到 ID2，只扫描一行就结束了。

所以在建立前缀索引的时候就需要根据区分度建立合适的索引（前面已经说明了怎么计算区分度）

如果使用 index1（即 email 整个字符串的索引结构）的话，可以利用覆盖索引，从 index1 查到结果后直接就返回了，不需要回到 ID 索引再去查一次。而如果使用 index2（即 email(6) 索引结构）的话，就不得不回到 ID 索引再去判断 email 字段的值。
即使你将 index2 的定义修改为 email(18) 的前缀索引，这时候虽然 index2 已经包含了所有的信息，但 InnoDB 还是要回到 id 索引再查一下，因为系统并不确定前缀索引的定义是否截断了完整信息。

# 前缀索引的优化

**倒序存储**

适合字段值前面部分重复度高，后半部分重复度低，这时可以倒序存储数据。查询时可以这么写

```
mysql> select field_list from t where id_card = reverse('input_id_card_string');
```

**做 hash**

新增一个字段，专门存储字段的 hash 值：
```
mysql> alter table t add id_card_crc int unsigned, add index(id_card_crc);
```

新增一个字段，专门存储字段的 hash 值，每次插入数据时，都要调用 crc32() 这个函数得到校验码填到这个新字段。这个字段有可能重复，需要联合判断 id_card 的值是否精确相同。
```
mysql> select field_list from t where id_card_crc=crc32('input_id_card_string') and id_card='input_id_card_string'
```

二者的区别
1. 都不支持范围查询
2. hash 字段需要额外的空间
3. CPU 消耗：倒序插入时需要额外调用 reverse 函数，hash 需要调用 crc32() 函数。reverse 函数消耗的 CPU 更小一些；
4. hash 字段方式的查询效率更高，因为计算出来的 hash 值重复的可能性较小，扫描次数接近于 1

# 参考资料

极客时间的《mysql实战45讲》

