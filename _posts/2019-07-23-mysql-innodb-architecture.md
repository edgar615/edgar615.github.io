---
layout: post
title: InnoDB架构-简介（part1）
date: 2019-07-23
categories:
    - MySQL
comments: true
permalink: innodb-architecture.html
---

![](/assets/images/posts/innodb-architecture/innodb-architecture-1.png)

InnoDB 的架构分为两块：内存中的结构和磁盘上的结构。InnoDB 使用日志先行策略，将数据修改先在内存中完成，并且将事务记录成重做日志(Redo Log)，转换为顺序IO高效的提交事务。这里日志先行，说的是日志记录到数据库以后，对应的事务就可以返回给用户，表示事务完成。但是实际上，这个数据可能还只在内存中修改完，并没有刷到磁盘上去。内存是易失的，如果在数据落地前，机器挂了，那么这部分数据就丢失了。
InnoDB 通过 redo 日志来保证数据的一致性。如果保存所有的重做日志，显然可以在系统崩溃时根据日志重建数据。当然记录所有的重做日志不太现实，所以 InnoDB 引入了检查点机制。即定期检查，保证检查点之前的日志都已经写到磁盘，则下次恢复只需要从检查点开始。

# InnoDB 内存中的结构

内存中的结构主要包括 Buffer Pool，Change Buffer、Adaptive Hash Index以及 Log Buffer 四部分。如果从内存上来看，Change Buffer 和 Adaptive Hash Index 占用的内存都属于 Buffer Pool，Log Buffer占用的内存与 Buffer Pool独立。

- [Buffer Pool](https://edgar615.github.io/innodb-buffer-pool.html)
- [Change Buffer](https://edgar615.github.io/innodb-change-buffer.html)
- [Log Buffer](https://edgar615.github.io/innodb-log-buffer.html)
- [自适应哈希索引](https://edgar615.github.io/mysql-index-adaptive-hash-index.html)

# InnoDB 磁盘上的结构

磁盘中的结构分为两大类：表空间和重做日志。

- [表空间](https://edgar615.github.io/innodb-tablespace.html)：分为系统表空间(MySQL 目录的 ibdata1 文件)，临时表空间，常规表空间，Undo 表空间以及 file-per-table 表空间(MySQL5.7默认打开file_per_table 配置）。系统表空间又包括了InnoDB数据字典，[双写缓冲区(Doublewrite Buffer)](https://edgar615.github.io/innodb-doublewrite-buffer.html)，修改缓存(Change Buffer），Undo日志等。
- [Redo日志](https://edgar615.github.io/innodb-redo-log.html)：存储的就是 Log Buffer 刷到磁盘的数据。


# 总结

看完一系列文章之后，再看这个图就能够很清晰的理解innodb了

![](/assets/images/posts/innodb-architecture/innodb-architecture-2.png)



# 参考资料

https://www.jianshu.com/p/d4cc0ea9d097