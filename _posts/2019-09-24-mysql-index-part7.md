---
layout: post
title: 面试题：InnoDB 一棵 B+ 树可以存放多少行数据(part7)
date: 2019-09-24
categories:
    - MySQL
comments: true
permalink: mysql-index-btree-height.html
---

一个面试题：InnoDB 一棵 B+ 树可以存放多少行数据

> InnoDB 存储引擎也有自己的最小储存单元——页（Page），一个页的大小是 16K。假设一行数据的大小是 1K，那么一个页可以存放 16 行这样的数据。在 B+ 树中叶子节点存放数据，非叶子节点存放键值+指针。
> 可以通过`innodb_page_size`调整

我们先假设 B+ 树高为 2，即存在一个根节点和若干个叶子节点，那么这棵 B+ 树的存放总记录数为：根节点指针数*单个叶子节点记录行数。

单个叶子节点（页）中的记录数=16K/1K=16

我们假设主键 ID 为 bigint 类型，长度为 8 字节，而**指针大小在 InnoDB 源码中设置为 6 字节**，这样一共 14 字节。那么非叶子节点的记录数=16384字节/14=1170

> 16*1024 = 16384

那么可以算出一棵高度为 2 的 B+ 树，能存放 `1170*16=18720 `条这样的数据记录。

一棵高度为 3 的 B+ 树，能存放 `1170*1170*16=21902400` 条这样的数据记录。

>  所以在 InnoDB 中 B+ 树高度一般为 1-3 层，它就能满足千万级的数据存储。
> 在查找数据时一次页的查找代表一次 IO，所以通过主键索引查询通常只需要 1-3 次 IO 操作即可查找到数据。



# 怎么得到 InnoDB 主键索引 B+ 树的高度？

在 InnoDB 的表空间文件中，约定 page number 为 3 的代表主键索引的根页，而在根页偏移量为 64 的地方存放了该 B+ 树的 page level。

如果 page level 为 1，树高为 2，page level 为 2，则树高为 3。即 B+ 树的高度=page level+1；下面我们将从实际环境中尝试找到这个 page level。

```
SELECT
b.name, a.name, index_id, type, a.space, a.PAGE_NO
FROM
information_schema.INNODB_SYS_INDEXES a,
information_schema.INNODB_SYS_TABLES b
WHERE
a.table_id = b.table_id AND a.space <> 0;
```

> 下面完全摘自参考资料

![](/assets/images/posts/mysql-index/mysql-index-3.png)

可以看出数据库 dbt3 下的 customer 表、lineitem 表主键索引根页的 page number 均为 3，而其他的二级索引 page number 为 4。

下面我们对数据库表空间文件做想相关的解析：

![](/assets/images/posts/mysql-index/mysql-index-4.png)

因为主键索引 B+ 树的根页在整个表空间文件中的第 3 个页开始，所以可以算出它在文件中的偏移量：16384*3=49152（16384 为页大小）。

另外根据《InnoDB 存储引擎》中描述在根页的 64 偏移量位置前 2 个字节，保存了 page level 的值。

因此我们想要的 page level 的值在整个文件中的偏移量为：16384*3+64=49152+64=49216，前 2 个字节中。

![](/assets/images/posts/mysql-index/mysql-index-5.png)

接下来我们用 hexdump 工具，查看表空间文件指定偏移量上的数据：

- linetem 表的 page level 为 2，B+ 树高度为page level+1=3。
- region 表的 page level 为 0，B+ 树高度为 page level+1=1。
- customer 表的 page level 为 2，B+ 树高度为 page level+1=3。

这三张表的数据量如下：

![](/assets/images/posts/mysql-index/mysql-index-5.png)

lineitem 表的数据行数为 600 多万，B+ 树高度为 3，customer 表数据行数只有 15 万，B+ 树高度也为 3。

可以看出尽管数据量差异较大，这两个表树的高度都是 3。换句话说这两个表通过索引查询效率并没有太大差异，因为都只需要做 3 次 IO。

那么如果有一张表行数是一千万，那么他的 B+ 树高度依旧是 3，查询效率仍然不会相差太大。region 表只有 5 行数据，当然他的 B+ 树高度为 1。

# 参考资料

https://mp.weixin.qq.com/s/BWlkrHiB-uP6fDnsxtKU0Q