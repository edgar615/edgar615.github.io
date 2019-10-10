---
layout: post
title: MyISAM锁
date: 2019-09-27
categories:
    - MySQL
comments: true
permalink: mysql-myisam-lock.html
---

Mysql中不同的存储引擎支持不同的锁机制。比如MyISAM和MEMORY存储引擎采用的表级锁，BDB采用的是页面锁，也支持表级锁，InnoDB存储引擎既支持行级锁，也支持表级锁，默认情况下采用行级锁。

Mysql3种锁特性如下：

- 表级锁：开销小，加锁块；不会出现死锁，锁定粒度大，发生锁冲突的概率最高，并发度最低。
- 行级锁：开销大，加锁慢；会出现死锁；锁定粒度最小，发生锁冲突的概率最低，并发性也最高。
- 页面锁：开销和加锁界于表锁和行锁之间，会出现死锁；锁定粒度界与表锁和行锁之间，并发一般。

MyISAM存储引擎只支持表锁，mysql的表锁有两种模式：读锁和写锁。

- myISAM表的读操作，不会阻塞其他用户对同一个表的读请求，但会阻塞对同一个表的写请求
- myISAM表的写操作，会阻塞其他用户对同一个表的读和写操作
- myISAM表的读、写操作之间、以及写操作之间是串行的。

MyISAM在执行查询语句（select）前，会自动给涉及的所有表加上读锁。在执行更新操作（update,delete,insert等）前，会自动给涉及的表加上写锁，这个过程不需要用户干预。也可以手动加锁

# 手动加锁

```
#锁定表
LOCK TABLES 
    tb_name1 [AS alias] {READ[LOCAL]|[LOW_PRIORITY] WRITE}
    tb_name2 [AS alias] {READ[LOCAL]|[LOW_PRIORITY] WRITE}
    ...
#释放表锁定
UNLOCK TABLES;
```

- lock tables 可以锁定用于当前线程（会话）的表。如果被其他线程锁定，则当前线程会等待，直到可以获取所有锁定为止。
- unlock tables释放当前线程（会话）获得的任何锁定。
- read(读锁/共享锁)：当表不存在 WRITE 写锁时 READ 读锁被执行,这该状态下,当前线程不可以修改(insert,update,delete),其他线程的修改操作进入列队,当当前线程释放锁,其他线程修改被执行。
- read local：read local和read之间的区别是，read local允许在锁定被保持时，执行非冲突性INSERT语句（同时插入）。但是，如果您正打算在MySQL外面操作数据库文件，同时您保持锁定，则不能使用read local。对于InnoDB表，read local与read相同。
- write(写锁/排它锁)：除了当前用户被允许读取和修改被锁表外，其他用户的所有访问（读/写）被完全阻止。注意的是在当前线程当WRITE被执行的时候,即使之前加了READ没被取消,也会被取消。
- low_priority write：降低优先级的write,默认write的优先级高于read.假如当前线程的low_priority write在列队里面,在未执行之前其他线程传送一条read,那么low_priority write继续等待.

说明：

- lock tables 加上‘local’选项，其作用就是在满足MyISAM表并发插入条件下，允许其他用户在表尾并发的插入记录。
- 在lock tables 给表显式加表锁时候，必须同时取得所有涉及表的锁，并且MySQL不支持锁升级。即在执行lock tables后，只能访问显式加锁的这些表，不能访问未加锁的表。MyISAM总是一次获取sql语句所需要的全部锁。这就是MyISAM表不会出现死锁的原因。
- 当使用lock tables时，不仅需要一次锁定用到的表，而且，同一个表在sql语句中出现多少次，就要在相同的别名中锁定多少次。

# 锁争用情况分析
通过检查`table_locks_waited`(表锁等待,无法立即获得数据)和`table_locks_immediate`(立即获得锁地查询数目)状态变量分析系统上表锁争夺情况。如果`table_locks_waited` 数值比较高,就说明存在着较严重的表级锁争用情况 ,性能有问题，并发高,需要优化.

```
mysql> show status like '%table_lock%';
+-----------------------------------------+----------+
| Variable_name                           | Value    |
+-----------------------------------------+----------+
| Performance_schema_table_lock_stat_lost | 0        |
| Table_locks_immediate                   | 22435132 |
| Table_locks_waited                      | 3        |
+-----------------------------------------+----------+
3 rows in set (0.03 sec)
```
# 锁调度

MyISAM存储引擎的读锁和写锁是互斥的，读写操作是串行的，那么如果读写两个进程同时请求同一张表，Mysql将会使写进程先获得锁。不仅仅如此，即使读请求先到达锁等待队列，写锁后到达，写锁也会先执行。因为mysql认为写请求比读请求更加重要。**这也正是MyISAM不适合含有大量更新操作和查询操作应用的原因。**因为大量的更新操作会造成查询操作很难获得读锁，从而可能永远阻塞。同时，一些需要长时间运行的查询操作，也会使写线程“饿死” ，应用中应尽量避免出现长时间运行的查询操作（在可能的情况下可以通过使用中间表等措施对SQL语句做一定的“分解” ，使每一步查询都能在较短时间完成，从而减少锁冲突。如果复杂查询不可避免，应尽量安排在数据库空闲时段执行，比如一些定期统计可以安排在夜间执行）。

我们可以通过一些设置来改变锁处理先后顺序：

- 通过启动参数`low-priority-updates`,使MyISAM引擎默认给予读侵权以优先权利
- 通过`set low_priority_updates=1`,使该连接发出的更新请求优先级降低
- 通过指定insert,update,delete语句的low_priority属性，降低该语句的优先级
- 还可以设置max_write_lock_count。当表的读锁到这个值的时候，mysql就暂时将写请求的优先级降低，给读进程一个获得锁的机会

# 并发插入
在一定的条件下，MyISAM表支持查询和插入并发执行。MyISAm有一个系统变量`concurrent_insert`，用来专门控制其并发行为的。

- concurrent_insert设置为0时，不允许并发插入
- 当concurrent_insert设置为1时，如果MyISAM表中没有空洞（即表的中间没有被删除的行），MyISAM允许在一个线程读表的同时，另一个线程从表尾插入记录。这也是MySQL的默认设置
- 当concurrent_insert设置为2时，无论MyISAM表中有没有空洞，都允许在表尾并发插入记录。

# 参考资料

https://www.hollischuang.com/archives/1728

https://blog.csdn.net/pursuing0my0dream/article/details/45166975