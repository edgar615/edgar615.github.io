---
layout: post
title: InnoDB架构-Change Buffer（part3）
date: 2019-07-23
categories:
    - MySQL
comments: true
permalink: innodb-change-buffer.html
---

通常来说，InnoDB辅助索引不同于聚集索引的顺序插入，如果每次修改二级索引都直接写入磁盘，则会有大量频繁的随机IO。

对于数据的修改需要分两种情况考虑

## 辅助索引已经在缓冲池

如果辅助索引页已经在缓冲池了，则直接修改即可，包含两次操作

1. 直接修改缓冲池的数据，一次内存操作
2. 顺序写redo log，一次磁盘顺序写（效率很高）

## 辅助索引不在缓冲池

如果辅助索引页不在缓冲池，在没有change buffer的情况下，包含三次操作

1. 把对应的索引页，从磁盘加载到缓冲池，一次磁盘随机读操作
2. 修改缓冲池的数据，一次内存操作
3. 顺序写redo log，一次磁盘顺序写

> 和聚集索引不同，辅助索引通常是不唯一的，插入辅助索引通常也是随机的。同样，对辅助索引的删除、更新也通常是不连续的。
> redo log是顺序写，会很快

# change buffer

![](/assets/images/posts/change-buffer/change-buffer-1.png)

change buffer是一个特殊的数据结构，当二级索引的页面不在缓冲池中，**change buffer会缓存对二级索引的数据操作（update、insert、delete）。主要是减少磁盘的随机I/O.**（仅支持二级索引，不支持聚集索引、全文索引、空间索引）。它会占用部分Buffer Pool 的内存空间。在 MySQL5.5 之前 Change Buffer其实叫 Insert Buffer，最初只支持 insert 操作的缓存，随着支持操作类型的增加，改名为 Change Buffer。Change Buffer 内部实现也是使用的 B+ 树。

对辅助索引页进行写操作时，如果页不在缓冲池，并不会立即将页加载到缓冲池，而是先将修改记录保存到 Change Buffer。等到未来Change Buffer数据对应的辅助索引页被读取到缓冲区时合并到真正的辅助索引页中。

加入写缓冲优化后，前面的流程优化为：

1. 在change buffer中记录这个操作，一次内存操作
2. 顺序写redo log，一次磁盘顺序写

如果之后读取对应的索引页，流程为：

1. 将索引页读取到缓冲池，一次磁盘IO
2. 从写缓冲读取相关数据
3. 恢复索引页，放到缓冲池

**唯一索引不会使用change buffer。** 如果索引设置了唯一(unique)属性，在进行插入或修改操作时，InnoDB必须进行唯一性检查。如果不读取索引页到缓冲池中，无法校验索引是否唯一。**但是可以缓存删除操作**

## 清除change buffer的操作
下面几种情况会导致purge(清除)change buffer的操作

- 用户线程选择辅助索引进行数据查询，这时候必须要读入辅助索引页，相应的ibuf entry需要merge到Page中。之后该page会被刷新到磁盘
- 当系统空闲或者slow shutdown时，后台master线程发起merge
- change buffer 页面没有空间了**change buffer默认占有buffer pool内存的25%，最大为50%。**

## 适应场景
不适合的场景：

1. 数据库都是唯一索引
2. 写入一个数据后，会立刻读取它；

这两类场景，在写操作进行时（进行后），本来就要进行进行页读取，本来相应页面就要入缓冲池，此时写缓存反倒成了负担，增加了复杂度。

适合的场景

1. 数据库大部分是非唯一索引；
2. 业务是写多读少，或者不是写后立刻读取；

可以使用写缓冲，将原本每次写入都需要进行磁盘IO的SQL，优化定期批量写磁盘。

# 配置参数

`innodb_change_buffer_max_size%` 配置写缓冲的大小，占整个缓冲池的比例，默认值是25%，最大值是50%。

```
mysql> show variables like '%innodb_change_buffer_max_size%';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| innodb_change_buffer_max_size | 25    |
+-------------------------------+-------+
1 row in set (0.03 sec)
```

`innodb_change_buffering`配置是否缓存辅助索引页的修改，默认为 all，即缓存 insert/delete-mark/purge 操作(注：MySQL 删除数据通常分为两步，第一步是delete-mark，即只标记，而purge才是真正的删除数据)。

```
mysql> show variables like '%innodb_change_buffering%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| innodb_change_buffering | all   |
+-------------------------+-------+
1 row in set (0.04 sec)
```

# 参考资料

https://mp.weixin.qq.com/s/PF21mUtpM8-pcEhDN4dOIw