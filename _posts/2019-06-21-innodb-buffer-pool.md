---
layout: post
title: InnoDB内存（1）- Buffer Pool
date: 2019-06-21
categories:
    - MySQL
comments: true
permalink: innodb-buffer-pool.html
---

# 1. 缓冲池buffer pool

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

> 缓冲池缓存的数据包括Page Cache、Change Buffer、Data Dictionary Cache等，通常 MySQL 服务器的 **80%** 的物理内存会分配给 Buffer Pool
>
> InnoDB内存中的结构主要包括 Buffer Pool，Change Buffer、Adaptive Hash Index以及 Log Buffer 四部分。如果从内存上来看，Change Buffer 和 Adaptive Hash Index 占用的内存都属于 Buffer Pool，Log Buffer占用的内存与 Buffer Pool独立

# 2. 预读

InnoDB在I/O的优化上有个比较重要的特性为预读，预读请求是一个i/o请求，它会异步地在缓冲池中预先回迁多个页面，预计很快就会需要这些页面，这些请求在一个范围内引入所有页面。

数据库请求数据的时候，会将读请求交给文件系统，放入请求队列中；相关进程从请求队列中将读请求取出，根据需求到相关数据区(内存、磁盘)读取数据；取出的数据，放入响应队列中，最后数据库就会从响应队列中将数据取走，完成一次数据读操作过程。接着进程继续处理请求队列，(如果数据库是全表扫描的话，数据读请求将会占满请求队列)，判断后面几个数据读请求的数据是否相邻，再根据自身系统IO带宽处理量，进行预读，进行读请求的合并处理，一次性读取多块数据放入响应队列中，再被数据库取走。(如此，一次物理读操作，可以实现多页数据读取)

InnoDB使用两种预读算法来提高I/O性能：**线性预读（linear read-ahead）**和**随机预读（randomread-ahead）**，线性预读着眼于将下一个区提前读取到buffer pool中，而随机预读着眼于将当前区中的剩余的页提前读取到buffer pool中。

## 2.1. 线性预读
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

## 2.2. 随机预读
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

## 2.3. 监控InnoDB预读
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
# 3. MySQL缓冲池污染

当某一个SQL语句，要批量扫描大量数据时，可能导致把缓冲池的所有页都替换出去，导致大量热数据被换出，MySQL性能急剧下降，这种情况叫缓冲池污染。

如果某个表中记录非常多的话，那该表会占用特别多的页，当对这个表执行全表扫描的时候会将大量的页加载到Buffer Pool中，这也就意味着需要把Buffer Pool中的所有页都替换一次，而这时其他查询语句在执行时又得执行一次从磁盘加载到Buffer Pool的操作。而这种全表扫描的语句执行的频率也不高，每次执行都要把Buffer Pool中的缓存页替换一次，这严重的影响到其他查询对 Buffer Pool的使用，从而大大降低了缓存命中率。

# 4. LRU List、Free List和Flush List

![](/assets/images/posts/mysql-buffer/innodb-buffer-pool-2.png)

**Free链表**用于记录缓冲池中哪些缓存页是空闲的。

**Flush链表**用来记录缓冲池中哪些缓存页是**脏页**

**LRU链表** 缓冲池的大小毕竟有限，不可能无限增长，如果free链表中已经没有多余的空闲缓存页，就需要把某些旧的缓存页从缓冲池中移除，然后再加入新的页。当缓冲池中不再有空闲的缓存页时，就需要淘汰掉部分最近很少使用的缓存页。**LRU链表**就是用来按照最近最少使用的原则去淘汰缓存页的，当我们需要访问某个页时，就把该缓存页调整到LRU链表的头部，这样LRU链表尾部就是最近最少使用的缓存页

> **空闲页** Free Page 缓存页未被使用
>
> **干净页** Clean Page 缓存页已被使用但是页面未发生修改
>
> **脏页**  如果我们修改了缓冲池中某个缓存页的数据，那它就和磁盘上的页不一致了，这样的缓存页也被称为脏页（dirty page）。如果发生一次修改就立即同步到磁盘上对应的页上会严重的影响程序的性能。所以需要一个链表来记录哪些缓存页需要被刷新到磁盘上

## 4.1. LRU链表

根据前面介绍可以知道某些SQL操作可能会使缓冲池中的页被刷新出，从而影响缓冲池的效率。例如作为索引或者数据的扫描操作，这类操作需要访问表中的许多页，甚至是全部的页，而这些页通常来说仅仅在这次查询操作中需要，并不是活跃的热点数据。如果页被放入LRU链表的首部，那么非常可能需要的热点数据页从LRU链表中移除，而在下一次需要读取该页时，InnoDB需要再次访问磁盘。

所以InnoDB对缓冲池的LRU算法做了改进 **LRU列表中加入了一个midpoint位置。新读取到的页，虽然是最新访问的页，但并不是直接放入到LUR列表的首部，而是放入到LRU列表的midpoint位置。这就是所谓的中点插入策略**

一般情况下list 头部存放都是最为活跃的热点数据，就是所谓的young page（最近经常访问的数据），list尾部存放的就是old page（最近不被访问的数据）。**这个算法就保证了**最近经常使用的page信息会被保存在最近访问的sublist，相反的不被经常访问的就会保存在old sublist。而old sublist当中的page信息都是在新数据写入时被驱逐的。

![](/assets/images/posts/mysql-buffer/innodb-buffer-pool-list.png)

可以通过`innodb_old_blocks_pct`控制old和new的分布，默认值是37（`3/8*100`）。该值取值范围为5~95，为全局动态变量。

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
- 频繁访问一个Buffer Pool的缓存页，会促使页面往Young链表的头部移动。如果一个Page在被读到Buffer Pool后很快就被访问，那么该Page会往Young List的头部移动，但是如果一个页面是通过预读的方式读到Buffer Pool，且之后短时间内没有被访问，那么很可能在下次访问之前就被移动到Old List的尾部，而被驱逐了。
- 随着数据库的持续运行，新的页面被不断的插入到LRU链表的Mid Point，Old 链表里的页面会逐渐的被移动Old链表的尾部。同时，当经常被访问的页面移动到LRU链表头部的时候，那些没有被访问的页面会逐渐的被移动到链表的尾部。最终，位于Old 链表尾部的页面将被驱逐。

一般情况下，页信息会被查询语句立马查询到而被移动到new sublist，这就意味着他们会在Buffer Pool里面保留很长一段时间。

如果一个数据页已经处于Young 链表，当它再次被访问的时候，只有当其处于Young 链表长度的1/4(大约值)之后，才会被移动到Young 链表的头部。这样做的目的是减少对LRU 链表的修改，因为LRU 链表的目标是保证经常被访问的数据页不会被驱逐出去。

表扫描（包括mysqldump或者没有where条件的select等操作）等操作将会刷入大量的数据进入Buffer Pool，同时也会将更多的Buffer Pool当中的信息刷出去，即使这个操作可能只会使用到一次而已。同样的如果 read-ahead后台进程读入大量数据的情况下也是会造成Buffer Pool大量高频的刷新数据页，但是这些操作是可控的。

>  全表扫描有一个特点，那就是它的执行频率非常低，而且在执行全表扫描的过程中，即使某个页面中有很多条记录，也就是去多次访问这个页面所花费的时间也是非常少的。

InnoDB通过`innodb_old_blocks_time`控制的Old 链表头部页面的转移策略。该Page需要在Old 链表停留超过`innodb_old_blocks_time` 时间，之后再次被访问，才会移动到Young 链表。这么操作是避免Young 链表被那些只在`innodb_old_blocks_time`时间间隔内频繁访问，之后就不被访问的页面塞满，从而有效的保护Young 链表。默认值1000毫秒

```
mysql> show variables like 'innodb_old_blocks_time'\G
*************************** 1. row ***************************
Variable_name: innodb_old_blocks_time
        Value: 1000
1 row in set (0.00 sec)
```

LRU列表用了管理已经读取的页，但当数据库刚启动时，LRU列表是空的，即没有任何页，这时页都存放在Free列表中。当需要从缓冲池中分页时，首先从free列表中查找是否有可用的空闲页，若有则将该页从free列表中移除放入到LRU列表中，否则根据LRU算法，淘汰LRU列表尾部的页，将该内存空间分片给新的页。当页从LRU的old部分加入到new部分时，称此时发生的操作为page made young，而因为`innodb_old_blocks_time`的设置导致页每页从old部分移动到new部分的操作称为page not made young。

**在数据库的Buffer Pool里面，不管是new列表还是old列表的数据如果不会被访问到，最后都会被移动到list的尾部作为被淘汰的页。**

调大`innodb_old_blocks_time`提高了从Old链表移动到Young链表的难度，会促使更多页面被移动到Old 链表，老化，从而被驱逐。

当扫描的表很大，Buffer Pool都放不下时，可以将innodb_old_blocks_pct设置为较小的值，这样只读取一次的数据页就不会占据大部分的Buffer Pool。例如，设置`innodb_old_blocks_pct = 5`，会将仅读取一次的数据页在Buffer Pool的占用限制为5％。

当经常扫描一些小表时，这些页面在Buffer Pool移动的开销较小，我们可以适当的调大innodb_old_blocks_pct，例如设置`innodb_old_blocks_pct = 50`。

## 4.2. Flush 链表

- Flush 链表里面保存的都是脏页，也会存在于LRU 链表
- Flush 链表是按照oldest_modification排序，值大的在头部，值小的在尾部
- 当有页面访被修改的时候，使用mini-transaction，对应的page进入Flush 链表
- 如果当前页面已经是脏页，就不需要再次加入Flush list，否则是第一次修改，需要加入Flush 链表
-  当Page Cleaner线程执行flush操作的时候，从尾部开始scan，将一定的脏页写入磁盘，推进检查点，减少recover的时间

## 4.3. Free 链表

- Free 链表 存放的是空闲页面，初始化的时候申请一定数量的页面
- 在执行SQL的过程中，每次成功load 页面到内存后，会判断Free 链表的页面是否够用。如果不够用的话，就flush LRU 链表和Flush 链表来释放空闲页。如果够用，就从Free 链表里面删除对应的页面，在LRU 链表增加页面，保持总数不变。

## 4.4. LRU 链表和Flush链表的区别

- LRU 链表 flush，由用户线程触发(MySQL 5.6.2之前)；而Flush 链表 flush由MySQL数据库InnoDB存储引擎后台srv_master线程处理。(在MySQL 5.6.2之后，都被迁移到Page Cleaner线程中)
- LRU 链表 flush，其目的是为了写出LRU 链表尾部的脏页，释放足够的空闲页，当Buffer Pool满的时候，用户可以立即获得空闲页面，而不需要长时间等待；Flush 链表 flush，其目的是推进Checkpoint LSN，使得InnoDB系统崩溃之后能够快速的恢复
- LRU 链表 flush，其写出的脏页，需要从LRU链表中删除，移动到Free 链表。Flush 链表 flush，不需要移动page在LRU链表中的位置
- LRU 链表 flush，每次flush的脏页数量较少，基本固定，只要释放一定的空闲页即可；Flush 链表 flush，根据当前系统的更新繁忙程度，动态调整一次flush的脏页数量，量很大
- 在Flush 链表上的页面一定在LRU 链表上，反之则不成立。

## 4.5. 触发刷脏页的条件

- REDO日志快用满的时候。由于MySQL更新是先写REDO日志，后面再将数据Flush到磁盘，如果REDO日志对应脏数据还没有刷新到磁盘就被覆盖的话，万一发生Crash，数据就无法恢复了。此时会从Flush 链表里面选取脏页，进行Flush。
- 为了保证MySQL中的空闲页面的数量，Page Cleaner线程会从LRU 链表尾部淘汰一部分页面作为空闲页。如果对应的页面是脏页的话，就需要先将页面Flush到磁盘。
- MySQL中脏页太多的时候。`innodb_max_dirty_pages_pct` 表示的是Buffer Pool最大的脏页比例，默认值是75%，当脏页比例大于这个值时会强制进行刷脏页，保证系统有足够可用的Free Page。`innodb_max_dirty_pages_pct_lwm`参数控制的是脏页比例的低水位，当达到该参数设定的时候，会进行preflush，避免比例达到`innodb_max_dirty_pages_pct`来强制Flush，对MySQL实例产生影响。
- MySQL实例正常关闭的时候，也会触发MySQL把内存里面的脏页全部刷新到磁盘。

> 当Buffer Pool中的脏页占用比达到`innodb_max_dirty_pages_pct_lwm`的设定值的时候，就会自动将脏页清出Buffer Pool，这是为了保证Buffer Pool当中脏页的占有率，也是为了防止脏页占有率超过`innodb_max_dirty_pages_pct`的设定值，当脏页的占有率达到了`innodb_max_dirty_pages_pct`的设定值的时候，InnoDB就会强制刷新Buffer Pool Pages。

**InnoDB采用一种基于redo log的最近生成量和最近刷新频率的算法来决定冲洗速度。**这样的算法可以保证数据库的冲洗不会影响到数据库的性能，也能保证数据库Buffer Pool中的数据的脏数据的占用比。这种自动调节的方式还能够防止突然的并发redo变大，但是flush的时候将不能进行普通的IO读写操作。

我们知道InnoDB使用日志的方式是循环使用的，在重用前一个日志文件之前，InnoDB就会将这个日志这个日志记录相关的所有在Buffer Pool当中的数据刷新到磁盘，也就是所谓的sharp checkpoint，和sqlserver的checkpoint很像。当一个插入语句产生大量的redo信息，需要记录的日志当前redo  log文件不能够完全存储，也会写入到当前的redo 文件当中。当redo log当中的所有使用空间都被用完了的，就会触发 sharp  checkpoint，所以这个时候即使脏数据占有率没有达到innodb_max_dirty_pages_pct ，还是会进行刷新。**这种算法是经得住考验的，所以说千万不要随便设置，最好是默认值**。但是我们从中也就会知道为什么一个事物的redo信息只能记录在一个redo log文件当中了。

因为有这么多的好处，所以 innodb_adaptive_flushing的值默认就是true的。

Innodb 的策略是在运行过程中尽可能的多占用内存，因此未被使用的页面会很少。当我们读取的数据不在Buffer Pool里面时，就需要申请一个空闲页来存放。如果没有足够的空闲页时，就必须从LRU 链表的尾部淘汰页面。如果该页面是干净的，可以直接拿来用，如果是脏页，就需要进行刷脏操作，将内存数据Flush到磁盘。

所以，如果出现以下情况，是很容易影响MySQL实例的性能：

- 一个SQL查询的数据页需要淘汰的页面过多
- 实例是个写多型的MySQL，checkpoint跟不上日志产生量，会导致更新全部堵塞，TPS跌0。


> 上面有部分都是抄的https://mp.weixin.qq.com/s/O29wR4qiYAPQzBjSwO8FZg

## 4.6. 查看LRU使用情况

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

`LRU len: 7141, unzip_LRU len: 0` innodb存储引擎支持压缩页功能，即将原本16k的页压缩为1k、2k、4k、8k。所以对于非16k的页通过unzip_LRU

Buffer pool size 共有 8191个页，共`8191*16K`的缓冲池空间，Free buffers表示当前Free列表的页的数量。Database pages 表示LRU列表中页的数量。可能Free buffers+Database pages 可能并不等于Buffer pool size ，因为缓冲池中的页还可能会被分配给insert buffer、自适应哈希索引、锁信息(lock info)，数据字典信息等

**需要关注的值：**

- `youngs/s`：该指标表示的是每秒访问Old 链表中页面，使其移动到Young链表的次数。如果MySQL实例都是一些小事务，没有大表全扫描，且该指标很小，就需要调大innodb_old_blocks_pct 或者减小innodb_old_blocks_time，这样会使得Old List 的长度更长，Old页面被移动到Old List 的尾部消耗的时间会更久，那么就提升了下一次访问到Old List里面的页面的可能性。如果该指标很大，可以调小innodb_old_blocks_pct，同时调大innodb_old_blocks_time，保护热数据。
- `non-youngs/s`：该指标表示的是每秒访问Old 链表中页面，没有移动到Young链表的次数，因为其不符合innodb_old_blocks_time。如果该指标很大，一般情况下是MySQL存在大量的全表扫描。如果MySQL存在大量全表扫描，且这个指标又不大的时候，需要调大innodb_old_blocks_time，因为这个指标不大意味着全表扫描的页面被移动到Young 链表了，调大innodb_old_blocks_time时间会使得这些短时间频繁访问的页面保留在Old 链表里面。

每隔1秒钟，Page Cleaner线程执行LRU List Flush的操作，来释放足够的Free Page。`innodb_lru_scan_depth` 变量控制每个Buffer Pool实例每次扫描LRU List的长度，来寻找对应的脏页，执行Flush操作。

还可以通过查询`innodb_buffer_pool_stats`查看缓冲池的运行状态

```

mysql> show tables like '%buffer%';
+-----------------------------------------+
| Tables_in_information_schema (%BUFFER%) |
+-----------------------------------------+
| INNODB_BUFFER_PAGE                      |
| INNODB_BUFFER_PAGE_LRU                  |
| INNODB_BUFFER_POOL_STATS                |
+-----------------------------------------+
3 rows in set (0.00 sec)

```



```
mysql> select * from INNODB_BUFFER_POOL_STATS\G
*************************** 1. row ***************************
                         POOL_ID: 0
                       POOL_SIZE: 8192
                    FREE_BUFFERS: 3506
                  DATABASE_PAGES: 4676
              OLD_DATABASE_PAGES: 1716
         MODIFIED_DATABASE_PAGES: 0
              PENDING_DECOMPRESS: 0
                   PENDING_READS: 0
               PENDING_FLUSH_LRU: 0
              PENDING_FLUSH_LIST: 0
                PAGES_MADE_YOUNG: 2622
            PAGES_NOT_MADE_YOUNG: 67161
           PAGES_MADE_YOUNG_RATE: 0
       PAGES_MADE_NOT_YOUNG_RATE: 0
               NUMBER_PAGES_READ: 1054
            NUMBER_PAGES_CREATED: 3744
            NUMBER_PAGES_WRITTEN: 27805
                 PAGES_READ_RATE: 0
               PAGES_CREATE_RATE: 0
              PAGES_WRITTEN_RATE: 0
                NUMBER_PAGES_GET: 8941309
                        HIT_RATE: 0
    YOUNG_MAKE_PER_THOUSAND_GETS: 0
NOT_YOUNG_MAKE_PER_THOUSAND_GETS: 0
         NUMBER_PAGES_READ_AHEAD: 0
       NUMBER_READ_AHEAD_EVICTED: 0
                 READ_AHEAD_RATE: 0
         READ_AHEAD_EVICTED_RATE: 0
                    LRU_IO_TOTAL: 25
                  LRU_IO_CURRENT: 0
                UNCOMPRESS_TOTAL: 0
              UNCOMPRESS_CURRENT: 0
1 row in set (0.00 sec)
```

可以通过innodb_buffer_page_lru 观察每个LRU列表中每个页的具体信息。

观察unzip_LRU 列表中的页

```
select * from INNODB_BUFFER_PAGE_LRU where compressed_size <> 0;
```

观察脏页

```
select * from INNODB_BUFFER_PAGE_LRU where oldest_modification > 0;
```



# 5. buffer pool相关配置
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


# 6. 参考资料

《MySQL技术内幕  InnoDB存储引擎  第2版》

《MySQL 是怎样运行的：从根儿上理解 MySQL》

https://www.cnblogs.com/geaozhang/p/7397699.html

https://mp.weixin.qq.com/s/ZPXcsogmO9BkKMKNLQxNLA

https://mp.weixin.qq.com/s/nA6UHBh87U774vu4VvGhyw

https://mp.weixin.qq.com/s/O29wR4qiYAPQzBjSwO8FZg