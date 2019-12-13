---
layout: post
title: InnoDB架构-doublewrite buffer（part6）
date: 2019-05-31
categories:
    - MySQL
comments: true
permalink: innodb-doublewrite-buffer.html
---

InnoDB默认每页大小是16KB，其数据校验也是针对这16KB来计算的，将数据写入到磁盘是以页为单位进行操作的。操作系统的一页是4KB，写文件也是以页作为单位操作的，那么每写一个InnoDB的page到磁盘上，操作系统需要写4个块。

![](/assets/images/posts/mysql-doublewrite-buffer/doublewrite-buffer-1.png)

而计算机硬件和操作系统，在极端情况下（比如断电）往往并不能保证这一操作的原子性，16K的数据，写入4K时，发生了系统断电或系统崩溃，只有一部分写是成功的，这种情况下就是partial page write（部分页写入）问题。这时页数据出现不一样的情形，从而形成一个"断裂"的也，使数据产生混乱。这个时候InnoDB对这种块错误是无能为力的.

![](/assets/images/posts/mysql-doublewrite-buffer/doublewrite-buffer-2.png)

> MySQL也无法根据redo log进行恢复，而MySQL在恢复的过程中是检查page的checksum，checksum就是pgae的最后事务号，发生partial page write问题时，page已经损坏，找不到该page中的事务号，就无法恢复。
> redo日志修复的前提是“页数据正确”并且redo日志正常。

为了解决这个问题，InnoDB使用了一种叫做doublewrite的特殊文件flush技术，在把页写到数据文件之前，InnoDB先把它们写到一个叫doublewrite buffer的连续区域内，在写doublewrite buffer完成后，InnoDB才会把页写到数据文件的适当的位置。如果在写页的过程中发生意外崩溃，InnoDB在稍后的恢复过程中在doublewrite buffer中找到完好的页副本用于恢复。

# doublewrite
doublewrite由两部分组成，一部分为内存中的doublewrite buffer，其大小为2MB，另一部分是磁盘上共享表空间(ibdata x)中连续的128个页，即2个区(extent)，大小也是2M。

![](/assets/images/posts/mysql-doublewrite-buffer/doublewrite-buffer-3.png)

1. 当一系列机制触发数据缓冲池中的脏页刷新时，并不直接写入磁盘数据文件中，而是先拷贝至内存中的doublewrite buffer中；
2. 接着从doublewrite buffer分两次写入磁盘共享表空间中(连续存储，顺序写，性能很高)，每次写1MB；
3、待第二步完成后，再将doublewrite buffer中的脏页数据写入实际的各个表空间文件(离散写)；(脏页数据固化后，即进行标记对应doublewrite数据可覆盖)

如果第二步失败，数据不会写到磁盘，磁盘中仍然是原始数据，innodb会载入磁盘的原始数据和redo日志比较，并重新刷到doublewrite buffer。

如果第三步失败，也就是发生partial page write的问题。恢复的时候先检查页内的checksum是否相同，如果不一致，那么innodb就不会通过redo日志来恢复了，而是直接刷新dw buffer中的数据。

> 为什么log write不需要doublewrite的支持？
> 因为redolog写入的单位就是512字节，也就是磁盘IO的最小单位，所以无所谓数据损坏。

# doublewrite的副作用

double write是一个buffer, 但其实它是开在物理文件上的一个buffer, 其实也就是file, 所以它会导致系统有更多的fsync操作, 而硬盘的fsync性能是很慢的, 所以它会降低mysql的整体性能。但是，doublewrite buffer写入磁盘共享表空间这个过程是连续存储，是顺序写，性能非常高，(约占写的10%)，牺牲一点写性能来保证数据页的完整还是很有必要的。

> 将数据从doublewrite buffer写到真正的segment中的时候，系统会自动合并连接空间刷新的方式，每次可以刷新多个pages。

# doublewrite参数

```
mysql> show global status like '%dblwr%';
+----------------------------+----------+
| Variable_name              | Value    |
+----------------------------+----------+
| Innodb_dblwr_pages_written | 30344257 |
| Innodb_dblwr_writes        | 2795294  |
+----------------------------+----------+
2 rows in set (0.03 sec)
```

- Innodb_dblwr_pages_written 记录写入DWB中页的数量。
- Innodb_dblwr_writes 记录DWB写操作的次数。

开启doublewrite后，每次脏页刷新必须要先写doublewrite，而doublewrite存在于磁盘上的是两个连续的区，每个区由连续的页组成，一般情况下一个区最多有64个页，所以一次IO写入应该可以最多写64个页。

我们可以观察`Innodb_dblwr_pages_written`与`Innodb_dblwr_writes`的比例，如果约等于64，那么说明系统的写压力非常大，有大量的脏页要往磁盘上写

# 参考资料

https://www.cnblogs.com/geaozhang/p/7241744.html