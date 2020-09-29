---
layout: post
title: 为什么MySQL的自增主键会不连续
date: 2019-06-30
categories:
    - MySQL
comments: true
permalink: mysql-id-nonsequence.html
---

在很多开发者的认知中，MySQL 的主键都应该是单调递增的，但是在我们与 MySQL 打交道的过程中会遇到两个问题，首先是记录的主键并不连续，其次是可能会创建多个主键相同的记录，我们将从以下的两个角度回答 MySQL 不单调和不连续的原因：

- 较早版本的 MySQL 将 AUTO_INCREMENT 存储在内存中，实例重启后会根据表中的数据重新设置该值；
- 获取 AUTO_INCREMENT 时不会使用事务锁，并发的插入事务可能出现部分字段冲突导致插入失败

# 1. 删除记录

AUTO_INCREMENT 属性虽然在 MySQL 中十分常见，但是在较早的 MySQL 版本中，它的实现还比较简陋，InnoDB 引擎会在内存中存储一个整数表示下一个被分配到的 ID，当客户端向表中插入数据时会获取 AUTO_INCREMENT 值并将其加一。

因为该值存储在内存中，所以在每次 MySQL 实例重新启动后，当客户端第一次向 table_name 表中插入记录时，MySQL 会使用如下所示的 SQL 语句查找当前表中 id 的最大值，将其加一后作为待插入记录的主键，并作为当前表中 AUTO_INCREMENT 计数器的初始值。

```
SELECT MAX(ai_col) FROM table_name FOR UPDATE;
```

但是如果**使用者不严格遵循关系型数据库的设计规范**，就会出现如下所示的数据不一致的问题：

```
insert order(buyer) values(..);
update order set buyer = 10 where id = 1;
delete from user where id = 10;
// 重启MySQL
insert user(name) values(..);
```

因为重启了 MySQL 的实例，所以内存中的 AUTO_INCREMENT 计数器会被重置成表中的最大值，当我们再向表中插入新的 user 记录时会重新使用 10作为主键，主键也就不是单调的了。在新的 user 记录插入之后，**order表中的记录就错误的引用了新的user** ，这其实是一个比较严重的错误。

然而这也不完全是 MySQL 的问题，如果我们严格遵循关系型数据库的设计规范，使用外键处理不同表之间的联系，就可以避免上述问题，因为当前 trades 记录仍然有外部的引用，所以外键会禁止 trades 记录的删除，不过多数公司内部的 DBA 都不推荐或者禁止使用外键，所以确实存在出现这种问题的可能。

然而在 MySQL 8.0 中，AUTO_INCREMENT 计数器的初始化行为发生了改变，每次计数器的变化都会写入到系统的重做日志（Redo log）并在每个检查点存储在引擎私有的系统表中。

> In MySQL 8.0, this behavior is changed. The current maximum auto-increment counter value is written to the redo log each time it changes and is  saved to an engine-private system table on each checkpoint. These  changes make the current maximum auto-increment counter value persistent across server restarts.

当 MySQL 服务被重启或者处于崩溃恢复时，它可以从持久化的检查点和重做日志中恢复出最新的 AUTO_INCREMENT计数器，避免出现不单调的主键也解决了这里提到的问题。

# 2. 并发事务

为了提高事务的吞吐量，MySQL 可以处理并发执行的多个事务，但是如果并发执行多个插入新记录的 SQL 语句，可能会导致主键的不连续。事务 1 向数据库中插入 id = 10 的记录，事务 2 向数据库中插入 id = 11和 id = 12 的两条记录。

不过如果在最后事务 1 由于插入的记录发生了唯一键冲突导致了回滚，而事务 2 没有发生错误而正常提交，在这时我们会发现当前表中的主键出现了不连续的现象，后续新插入的数据也不再会使用 10 作为记录的主键。

这个现象背后的原因也很简单，虽然在获取 AUTO_INCREMENT 时会加锁，但是该锁是语句锁，它的目的是保证 AUTO_INCREMENT 的获取不会导致线程竞争，而不是保证 MySQL 中主键的连续。

上述行为是由 InnoDB 存储引擎提供的 innodb_autoinc_lock_mode 配置控制的，该配置决定了获取 AUTO_INCREMENT 计时器时需要先得到的锁，该配置存在三种不同的模式，分别是传统模式（Traditional）、连续模式（Consecutive）和交叉模式（Interleaved），其中 MySQL 使用连续模式作为默认的锁模式（参考自增锁的文章）

这三种模式都不能解决 MySQL 自增主键不连续的问题，想要解决这个问题的终极方案是串行执行所有包含插入操作的事务，也就是使用数据库的最高隔离级别 ——  可串行化（Serialiable）。当然直接修改数据库的隔离级别相对来说有些简单粗暴，基于 MySQL  或者其他存储系统实现完全串行的插入也可以保证主键**在插入时的连续**，但是仍然不能避免删除数据导致的不连续。



# 3.参考资料

https://mp.weixin.qq.com/s/RUZHl9cU642FO-edS4utLA