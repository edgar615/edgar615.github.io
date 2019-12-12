---
layout: post
title: InnoDB架构-缓冲池（part2）
date: 2019-06-20
categories:
    - MySQL
comments: true
permalink: innodb-buffer-pool.html
---

# 缓冲池buffer pool

InnoDB是基于磁盘存储的，并将其中的记录按照页的方式进行管理。

在数据库系统中，由于CPU速度与磁盘速度之间的鸿沟，基于磁盘的数据库系统通常是由缓冲池技术来提高数据库的整体性能。缓冲池简单来说就是一块内存区域，通过内存的速度来弥补磁盘速度较慢对数据库性能的影响。

在数据库中进行读取页的操作，首先将从磁盘读到的页存放在缓冲池中，这个过程称为将页“FIX”在缓冲池中。下次再读取相同的页时，受限判断该页是否在缓冲池中，若在缓冲池中，称该页在缓冲池中被命中，直接读取该页。否则读取磁盘上的页。

> **局部性原理**：当一个数据被用到时，其附近的数据也通常会马上被使用
> 
> **磁盘预读**：磁盘的存取速度往往是主存的几百分分之一，因此为了提高效率，要尽量减少磁盘I/O。为了达到这个目的，磁盘往往不是严格按需读取，而是每次都会预读，即使只需要一个字节，磁盘也会从这个位置开始，顺序向后读取一定长度的数据放入内存
> 
> InnoDB存储引擎在处理客户端的请求时，当需要访问某个页的数据时，就会把完整的页的数据全部加载到内存中，也就是说即使我们只需要访问一个页的一条记录，那也需要先把整个页的数据加载到内存中。

对数据库中页的修改操作，则首先修改在缓冲池中的页，然后再以一定的频率刷新到磁盘上，这里需要注意的是，页从缓冲池刷新回磁盘的操作并不是在每次页发生变更时触发，而是通过一种称为CheckPoint的机制刷新回磁盘，同样这也是为了提高数据库的整体性能。

综上所述，缓冲池的大小直接影响着数据库的整体性能。InnoDB的缓冲池配置可以通过参数`innodb_buffer_pool_size来设置`

```
mysql> show variables like 'innodb_buffer_pool_size'\G
*************************** 1. row ***************************
Variable_name: innodb_buffer_pool_size
        Value: 134217728
1 row in set (0.01 sec)
```

 缓冲池中缓存的数据页类型有：索引页，数据页，undo页，插入缓冲(insert buffer)、自适应哈希索引(adaptive hash index)、InnoDB存储的锁信息(lock info)，数据字典信息(data dirctionary)等。索引页和数据页只是占缓冲池很大的一部分而已

InnoDB允许有多个缓冲池实例，每个页根据哈希评价分片到不同的缓冲池实例中，这样做的好处是减少数据库内部的资源竞争，增加数据库的并发处理能力，可以通过`innodb_buffer_pool_instances`来进行配置

```
mysql> show variables like 'innodb_buffer_pool_instances'\G
*************************** 1. row ***************************
Variable_name: innodb_buffer_pool_instances
        Value: 1
1 row in set (0.00 sec)
```

也可以用`show engine innodb status\G`命令来观察缓冲池

Buffer Pool中默认的缓存页大小和在磁盘上默认的页大小是一样的，都是16KB。为了更好的管理这些在Buffer Pool中的缓存页，InnoDB为每一个缓存页都创建了一些控制信息，这些控制信息包括该页所属的表空间编号、页号、缓存页在Buffer Pool中的地址、链表节点信息、一些锁信息以及LSN信息等等

每个缓存页对应的控制信息占用的内存大小是相同的，这部分被放在 Buffer Pool 的前边，缓存页被存放到 Buffer Pool 后边。

> 缓冲池缓存的数据包括Page Cache、Change Buffer、Data Dictionary Cache等，通常 MySQL 服务器的 80% 的物理内存会分配给 Buffer Pool
>
> InnoDB内存中的结构主要包括 Buffer Pool，Change Buffer、Adaptive Hash Index以及 Log Buffer 四部分。如果从内存上来看，Change Buffer 和 Adaptive Hash Index 占用的内存都属于 Buffer Pool，Log Buffer占用的内存与 Buffer Pool独立

# LRU List、Free List和Flush List

**Free链表**用于记录缓冲池中哪些缓存页是空闲的。

**Flush链表**用来记录缓冲池中哪些缓存页是**脏页**

> **脏页**  如果我们修改了缓冲池中某个缓存页的数据，那它就和磁盘上的页不一致了，这样的缓存页也被称为脏页（dirty page）。如果发生一次修改就立即同步到磁盘上对应的页上会严重的影响程序的性能。所以需要一个链表来记录哪些缓存页需要被刷新到磁盘上

**LRU链表** 缓冲池的大小毕竟有限，不可能无限增长，如果free链表中已经没有多余的空闲缓存页，就需要把某些旧的缓存页从缓冲池中移除，然后再加入新的页。当缓冲池中不再有空闲的缓存页时，就需要淘汰掉部分最近很少使用的缓存页。**LRU链表**就是用来按照最近最少使用的原则去淘汰缓存页的，当我们需要访问某个页时，就把该缓存页调整到LRU链表的头部，这样LRU链表尾部就是最近最少使用的缓存页


# 预读

InnoDB在I/O的优化上有个比较重要的特性为预读，预读请求是一个i/o请求，它会异步地在缓冲池中预先回迁多个页面，预计很快就会需要这些页面，这些请求在一个范围内引入所有页面。

数据库请求数据的时候，会将读请求交给文件系统，放入请求队列中；相关进程从请求队列中将读请求取出，根据需求到相关数据区(内存、磁盘)读取数据；取出的数据，放入响应队列中，最后数据库就会从响应队列中将数据取走，完成一次数据读操作过程。接着进程继续处理请求队列，(如果数据库是全表扫描的话，数据读请求将会占满请求队列)，判断后面几个数据读请求的数据是否相邻，再根据自身系统IO带宽处理量，进行预读，进行读请求的合并处理，一次性读取多块数据放入响应队列中，再被数据库取走。(如此，一次物理读操作，可以实现多页数据读取)

InnoDB使用两种预读算法来提高I/O性能：**线性预读（linear read-ahead）**和**随机预读（randomread-ahead）**，线性预读着眼于将下一个区提前读取到buffer pool中，而随机预读着眼于将当前区中的剩余的页提前读取到buffer pool中。

## 线性预读
innodb通过参数`innodb_read_ahead_threshold`控制触发innodb执行预读操作的时间。如果一个区中的被顺序读取的页超过或者等于该参数变量时，Innodb将会异步的将下一个区读取到buffer pool中，`innodb_read_ahead_threshold`可以设置为0-64的任何值，默认值为56，值越高，访问模式检查越严格。

```
mysql> show variables like 'innodb_read_ahead_threshold';
+-----------------------------+-------+
| Variable_name               | Value |
+-----------------------------+-------+
| innodb_read_ahead_threshold | 56    |
+-----------------------------+-------+
1 row in set (0.03 sec)
```
InnoDB只有在顺序访问当前区中的56个页时才触发线性预读请求，将下一个区读到内存中。在没有该变量之前，当访问到区的最后一个页的时候，InnoDB会决定是否将下一个区放入到buffer pool中。

## 随机预读
同一个区中的一些页在buffer pool中发现时，InnoDB会将该区中的剩余页一并读到buffer pool中。InnoDB通过参数`innodb_random_read_ahead`控制随机预读的开启。

> 由于随机预读方式带来了一些不必要的复杂性，同时在性能也存在不稳定性，所以默认是关闭状态

```
mysql> show variables like 'innodb_random_read_ahead';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| innodb_random_read_ahead | OFF   |
+--------------------------+-------+
1 row in set (0.04 sec)
```

## 监控InnoDB预读
可以通过`show engine innodb status;`监控预读信息
```
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
```

通过`Innodb_buffer_pool_read_ahead`和`Innodb_buffer_pool_read_ahead_evicted`评估预读算法的有效性

- Innodb_buffer_pool_read_ahead：通过预读(后台线程)读入innodb buffer pool中数据页数
- Innodb_buffer_pool_read_ahead_evicted：通过预读来的数据页没有被查询访问就被清理的pages，无效预读页数

```
mysql> show global status like '%read_ahead%';
+---------------------------------------+--------+
| Variable_name                         | Value  |
+---------------------------------------+--------+
| Innodb_buffer_pool_read_ahead_rnd     | 0      |
| Innodb_buffer_pool_read_ahead         | 352999 |
| Innodb_buffer_pool_read_ahead_evicted | 0      |
+---------------------------------------+--------+
3 rows in set (0.07 sec)
```

# MySQL缓冲池污染

当某一个SQL语句，要批量扫描大量数据时，可能导致把缓冲池的所有页都替换出去，导致大量热数据被换出，MySQL性能急剧下降，这种情况叫缓冲池污染。

如果某个表中记录非常多的话，那该表会占用特别多的页，当对这个表执行全表扫描的时候会将大量的页加载到Buffer Pool中，这也就意味着需要把Buffer Pool中的所有页都替换一次，而这时其他查询语句在执行时又得执行一次从磁盘加载到Buffer Pool的操作。而这种全表扫描的语句执行的频率也不高，每次执行都要把Buffer Pool中的缓存页替换一次，这严重的影响到其他查询对 Buffer Pool的使用，从而大大降低了缓存命中率。

# LRU算法
根据前面介绍可以知道某些SQL操作可能会使缓冲池中的页被刷新出，从而影响缓冲池的效率。例如作为索引或者数据的扫描操作，这类操作需要访问表中的许多页，甚至是全部的页，而这些页通常来说仅仅在这次查询操作中需要，并不是活跃的热点数据。如果页被放入LRU链表的首部，那么非常可能需要的热点数据页从LRU链表中移除，而在下一次需要读取该页时，InnoDB需要再次访问磁盘。

所以InnoDB对缓冲池的LRU算法做了改进 **LRU列表中加入了一个midpoint位置。新读取到的页，虽然是最新访问的页，但并不是直接放入到LUR列表的首部，而是放入到LRU列表的midpoint位置。**

在innodb把midpoint之后的列表称为old列表，之前的列表称为new列表，可以简单的理解为new列表中的页都是最为活跃的热点数据。

![](/assets/images/posts/mysql-buffer/innodb-buffer-pool-list.png)

可以通过`innodb_old_blocks_pct`控制old和new的分布，默认值是37

```
mysql> show variables like 'innodb_old_blocks_pct'\G
*************************** 1. row ***************************
Variable_name: innodb_old_blocks_pct
        Value: 37
1 row in set (0.01 sec)
```

将LRU链表分为old和new之后，InnoDB按下面的逻辑处理

- 当磁盘上的某个页面在初次加载到Buffer Pool中的某个缓存页时，该缓存页会被放到old列表的头部。这样针对预读到Buffer Pool却不进行后续访问的页面就会被逐渐从old列表逐出，而不会影响young列表中被使用比较频繁的缓存页。
- 在对某个处在old列表的缓存页进行第一次访问时就在它对应的控制块中记录下来这个访问时间，如果后续的访问时间与第一次访问的时间在某个时间间隔内，那么该页面就不会被从old列表移动到new列表的头部，否则将它移动到new列表的头部

因为全表扫描有一个特点，那就是它的执行频率非常低，而且在执行全表扫描的过程中，即使某个页面中有很多条记录，也就是去多次访问这个页面所花费的时间也是非常少的。

InnoDB通过`innodb_old_blocks_time`参数来控制页读取到mid位置后需要等待多久才会被加入到LRU列表的热端。默认值1000毫秒

```
mysql> show variables like 'innodb_old_blocks_time'\G
*************************** 1. row ***************************
Variable_name: innodb_old_blocks_time
        Value: 1000
1 row in set (0.00 sec)
```

LRU列表用了管理已经读取的页，但当数据库刚启动时，LRU列表是空的，即没有任何页，这时页都存放在Free列表中。当需要从缓冲池中分页时，首先从free列表中查找是否有可用的空闲页，若有则将该页从free列表中移除放入到LRU列表中，否则根据LRU算法，淘汰LRU列表尾部的页，将该内存空间分片给新的页。当页从LRU的old部分加入到new部分时，称此时发生的操作为page made young，而因为`innodb_old_blocks_time`的设置导致页每页从old部分移动到new部分的操作称为page not made young。

在数据库的Buffer Pool里面，不管是new列表还是old列表的数据如果不会被访问到，最后都会被移动到list的尾部作为被淘汰的页。

# show engine innodb status
可以用`show engine innodb status\G`命令来观察LRU列表和free列表的使用情况和运行状态

```
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137428992
Dictionary memory allocated 3807133
Buffer pool size   8192
Free buffers       1024
Database pages     7141
Old database pages 2616
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 127464, not young 10579743
0.00 youngs/s, 0.00 non-youngs/s
Pages read 393131, created 205050, written 19241821
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 7141, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
```

`Total large memory allocated 137428992`分配给Innodb的内存大小

`Dictionary memory allocated 3807133`分配给Innodb数据字典的内存大小

`Buffer pool size   8192` innodb_buffer_pool的大小(page)

`Free buffers       1024` innodb_buffer_pool lru列表中的空闲页面数量

`Database pages     7141` innodb_buffer_pool lru列表中的非空闲页面数

`Old database pages 2616` innodb_buffer_pool old子列表的页面数量

`Modified db pages  0` innodb_buffer_pool 中脏页的数量

`Pending reads`和`Pending writes`显示了pending的reads 和writes

`Buffer pool hit rate 1000 / 1000...`显示了Innodb的buffer pool命中率，通常要保证在998/1000以上。如果没有，可考虑增大buffer pool size，以及优化你的查询

`Pages read ahead 0.00/s`显示了每秒线性预读的次数（读入）。

`evicted without access`显示了每秒读出的pages

`Random read ahead 0.00/s`显示了每秒随机预读的次数。

Buffer pool size 共有 8191个页，共8191*16K的缓冲池空间，Free buffers表示当前Free列表的页的数量。Database pages 表示LRU列表中页的数量。可能Free buffers+Database pages 可能并不等于Buffer pool size ，因为缓冲池中的页还可能会被分配给insert buffer、自适应哈希索引、锁信息(lock info)，数据字典信息等

# buffer pool相关配置
- innodb_buffer_pool_size：这个值是设置InnoDB Buffer Pool的总大小；
- innodb_buffer_pool_chunk_size：InnoDB Buffer Pool的执行单元 chunk size的大小。这里面有个关系要确定一下，最好按照这个设置 `innodb_buffer_pool_size=innodb_buffer_pool_chunk_size * innodb_buffer_pool_instances*N（N>=1）`;
- innodb_buffer_pool_instances：设置InnoDB Buffer Pool实例的个数，每一个实例都有自己独立的list管理Buffer Pool；
- innodb_old_blocks_pct：默认InnoDB Buffer Pool中点的位置，默认值是37，最大100，也就是我们所谓的3/8的位置，可以自己设置；
- innodb_old_blocks_time：设置保留在Buffer Pool里面的数据在插入时候没有被改变list位置的时候的保存时间；
- innodb_read_ahead_threshold：参数控制MySQL何时进行预读，也可以控制MySQL预读数据时候对于数据的敏感度，如果Buffer Pool里面存储的数据页的频繁值大于innodb_read_ahead_threshold的值，InnoDB就会启动一个异步的预读操作；
- innodb_random_read_ahead：默认是disabled，是控制预读方式的参数，开启的话将不使用线性预读而是使用随机预读；
- innodb_adaptive_flushing：指定是否动态自适应刷新脏页到盘，这个是MySQL根据负载自己决定的。不过还是尽量不要设置，让MySQL自己来管理自己；
- innodb_adaptive_flushing_lwm：关闭adaptive_flushing的话才会有用，用来标记redo log使用率的百分比的最低线，当达到这个值的时候就会刷新脏页，默认为10；
- innodb_flush_neighbors：控制是否刷新Buffer Pool脏页的脏数据的时候将同一区的脏数据页一同刷新，默认值为1；
- innodb_flushing_avg_loops：为InnoDB保存InnoDB Buffer Pool前几次的冲洗状态快照的迭代数，默认值为30，增大的话，冲洗就会变得缓慢。减小的话冲洗的频率就会变高；
- innodb_lru_scan_depth：控制LRU算法的一个参数，用来控制Buffer Pool后台进程page_cleaner 刷新脏页的位置；
- innodb_max_dirty_pages_pct：参数会让InnoDB Buffer Pool刷新数据而不让脏数据的百分比超过这个值；
- innodb_max_dirty_pages_pct_lwm：InnoDB会自动维护后台作业自动从Buffer Pool当中清除脏数据，当Buffer Pool中的脏页占用比 达到innodb_max_dirty_pages_pct_lwm的设定值的时候，就会自动将脏页清出Buffer Pool；
- innodb_buffer_pool_filename:指定文件名字；
- innodb_buffer_pool_dump_at_shutdown:配置的InnoDB是否保留当前的缓冲池的状态，以避免在服务器重新启动后，还要经历一个漫长的暖机时间；
- innodb_buffer_pool_load_at_startup：指定此参数启动，数据库重启以后会自动暖机，读入Buffer Pool重启前保存的信息；
- innodb_buffer_pool_dump_now和innodb_buffer_pool_load_now当数据库已经提起来的时候，我们忘了以前指定，也可以指定马上恢复；
- innodb_buffer_pool_dump_pct：设置一下恢复Buffer Pool中多少数据；
- innodb_buffer_pool_load_abort：终止Buffer Pool恢复，可以指定负载运行。

# 参考资料

《MySQL技术内幕  InnoDB存储引擎  第2版》

《MySQL 是怎样运行的：从根儿上理解 MySQL》

https://www.cnblogs.com/geaozhang/p/7397699.html

https://mp.weixin.qq.com/s/ZPXcsogmO9BkKMKNLQxNLA

https://mp.weixin.qq.com/s/nA6UHBh87U774vu4VvGhyw