---
layout: post
title: InnoDB架构（7） - doublewrite buffer
date: 2018-08-07
categories:
    - MySQL
comments: true
permalink: doublewrite-buffer.html
---

# 1. Double Write

InnoDB默认每页大小是16KB，其数据校验也是针对这16KB来计算的，将数据写入到磁盘是以页为单位进行操作的。操作系统的一页是4KB，写文件也是以页作为单位操作的，那么每写一个InnoDB的page到磁盘上，操作系统需要写4个块。

![](/assets/images/posts/mysql-doublewrite-buffer/doublewrite-buffer-1.png)

而计算机硬件和操作系统，在极端情况下（比如断电）往往并不能保证这一操作的原子性，16K的数据，写入4K时，发生了系统断电或系统崩溃，只有一部分写是成功的，这种情况下就是partial page write（部分页写入）问题。这时页数据出现不一样的情形，从而形成一个"断裂"的也，使数据产生混乱。这个时候InnoDB对这种块错误是无能为力的.

![](/assets/images/posts/mysql-doublewrite-buffer/doublewrite-buffer-2.png)

> MySQL也无法根据redo log进行恢复，而MySQL在恢复的过程中是检查page的checksum，checksum就是pgae的最后事务号，发生partial page write问题时，page已经损坏，找不到该page中的事务号，就无法恢复。
> redo日志修复的前提是“页数据正确”并且redo日志正常。

为了解决这个问题，InnoDB使用了一种叫做doublewrite的特殊文件flush技术，在把页写到数据文件之前，InnoDB先把它们写到一个叫doublewrite buffer的连续区域内，在写doublewrite buffer完成后，InnoDB才会把页写到数据文件的适当的位置。如果在写页的过程中发生意外崩溃，InnoDB在稍后的恢复过程中在doublewrite buffer中找到完好的页副本用于恢复。

**doublewrite由两部分组成，一部分为内存中的doublewrite buffer，其大小为2MB，另一部分是磁盘上共享表空间(ibdata x)中连续的128个页，即2个区(extent)，大小也是2M。**

![](/assets/images/posts/mysql-doublewrite-buffer/doublewrite-buffer-3.png)

1. 当一系列机制触发数据缓冲池中的脏页刷新时，并不直接写入磁盘数据文件中，而是先拷贝至内存中的doublewrite buffer中；
2. 接着从doublewrite buffer分两次写入磁盘共享表空间中(连续存储，顺序写，性能很高)，每次写1MB；
3. 待第二步完成后，再将doublewrite buffer中的脏页数据写入实际的各个表空间文件(离散写)；(脏页数据固化后，即进行标记对应doublewrite数据可覆盖)

如果第二步失败，数据不会写到磁盘，磁盘中仍然是原始数据，innodb会载入磁盘的原始数据和redo日志比较，并重新刷到doublewrite buffer。

如果第三步失败，也就是发生partial page write的问题。恢复的时候先检查页内的checksum是否相同，如果不一致，那么innodb就不会通过redo日志来恢复了，而是直接刷新dw buffer中的数据。

> 为什么log write不需要doublewrite的支持？
> 因为redolog写入的单位就是512字节，也就是磁盘IO的最小单位，所以无所谓数据损坏。

**doublewrite的副作用**

double write是一个buffer, 但其实它是开在物理文件上的一个buffer, 其实也就是file, 所以它会导致系统有更多的fsync操作, 而硬盘的fsync性能是很慢的, 所以它会降低mysql的整体性能。但是，doublewrite buffer写入磁盘共享表空间这个过程是连续存储，是顺序写，性能非常高，(约占写的10%)，牺牲一点写性能来保证数据页的完整还是很有必要的。

> 将数据从doublewrite buffer写到真正的segment中的时候，系统会自动合并连接空间刷新的方式，每次可以刷新多个pages。

# 2. 参数

MySQL8.0.20之前，doublewrite buffer存储在InnoDB 系统表空间中。从MySQL8.0.20开始，doublewrite buffer存储在双写文件中。

```

mysql> show variables like '%double%';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| innodb_doublewrite            | ON    |
| innodb_doublewrite_batch_size | 0     |
| innodb_doublewrite_dir        |       |
| innodb_doublewrite_files      | 2     |
| innodb_doublewrite_pages      | 4     |
+-------------------------------+-------+
5 rows in set (0.00 sec)
```

**`innodb_doublewrite`** 

`innodb_doublewrite` 控制是否启用doublewrite buffer。多数场景下默认是启用的。为了禁用doublewrite buffer，设置`innodb_doublewrite=0`或者启动MySQL服务时加`--skip-innodb-doublewrite`选项。例如在性能压测的场景下，如果您更关注性能而不是数据可靠性，您可以禁用doublewrite buffer。

如果doublewrite buffer位于支持原子写的Fusion-io设备上，则自动禁用doublewrite buffer，并使用Fusion-io原子写来执行数据文件写。但是，要注意`innodb_doublewrite`设置是全局的。当doublewrite buffer被禁用时，它对不在Fusion-io硬件上的数据文件也禁用。该功能仅在Fusion-io硬件上支持，在Linux中仅支持Fusion-io NVMFS。为了充分利用这个功能，建议将`innodb_flush_method=O_DIRECT`

**`innodb_doublewrite_dir`**

`innodb_doublewrite_dir`（8.0.20引入）定义了InnoDB创建双写文件的目录。如果目录没有指定，双写文件创建在`innodb_data_home_dir`目录下，没有指定默认在数据目录下。

哈希符'#'会自动创建在指定目录名前缀，避免与shema名冲突。然而，如果使用了'.', '#'. 或者'/'指定了目录前缀，则不在目录名前缀没有哈希符'#'。

理想情况下，双写目录应该放在最快的存储上。

**`innodb_doublewrite_files`**

`innodb_doublewrite_files`参数定义了双写文件的数量。默认情况下，每个缓冲池实例都会创建2个双写文件：一个刷新列表双写文件和一个LRU列表双写文件。

刷新列表双写文件用于从缓冲池刷新列表中刷新页。刷新列表双写文件默认大小是`InnoDB page size * doublewrite page bytes`.

LRU列表双写文件是用于刷新从缓冲池LRU列表的页。它也包括单个页刷新的槽。LRU列表双写文件默认大小为`InnoDB page size * (doublewrite pages + (512 / the number of buffer pool instances))`，512是为单个页刷新保留的槽的总数。

至少有2个双写文件。双写文件的最大数量是缓冲池实例的两倍。（缓冲池实例的数量由参数`innodb_buffer_pool_instances`控制）

双写文件有以下格式：`#ib_page_size_file_number.dblwr`。例如，下面的双写文件是在一个InnoDB页大小为16KB，单个缓冲池的MySQL实例上创建：

```
# ls *.dblwr
'#ib_16384_0.dblwr'  '#ib_16384_1.dblwr'
```

`innodb_doublewrite_files`参数用于高级性能调优。默认设定已经适用于大多数用户。

**`innodb_doublewrite_pages`**

`innodb_doublewrite_pages`参数（MySQL8.0.20引入）控制每个线程双写页的最大数量。如果这个值没有指定，`innodb_doublewrite_pages`设置为`innodb_write_io_threads`值。这个参数用于高级性能调优。默认值已经适用于大多数用户。

**`innodb_doublewrite_batch_size`**

`innodb_doublewrite_batch_size`参数（MySQL8.0.20引入）控制一批写入双写页的数量。这个参数用于高级性能调优。默认值已经适用于大多数用户。

# 3. 查看doublewrite状态

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

# 4. 参考资料

https://www.cnblogs.com/geaozhang/p/7241744.html

https://mp.weixin.qq.com/s/zJfpI3kH4NuE5mUKJwl2Fw