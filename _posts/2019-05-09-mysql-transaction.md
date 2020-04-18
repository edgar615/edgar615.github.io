---
layout: post
title: MySQL事务(part1)
date: 2019-05-09
categories:
    - MySQL
comments: true
permalink: mysql-transaction.html
---

事务是应用程序中一系列严密的操作，所有操作必须成功完成，否则在每个操作中所作的所有更改都会被撤消。也就是事务具有原子性，一个事务中的一系列的操作要么全部成功，要么一个都不做。

事务的结束有两种，当事务中的所以步骤全部成功执行时，事务提交。如果其中一个步骤失败，将发生回滚操作，撤消撤消之前到事务开始时的所以操作。

# ACID属性

事务具有四个特征：原子性（ Atomicity ）、一致性（ Consistency ）、隔离性（ Isolation ）和持续性（ Durability ）。这四个特性简称为 ACID 特性。

**原子性（Atomicity）：**事务中包含的操作集合，要么全部操作执行完成，要么全部都不执行。即当事务执行过程中，发生了某些异常情况，如系统崩溃、执行出错，则需要对已执行的操作进行回滚，清除所有执行痕迹。

 **一致性（Consistency）：**事务执行前和事务执行后，数据库的完整性约束不被破坏。即事务的执行是从一个有效状态转移到另一个有效状态。

 **隔离性（Isolation）：**多个事务并发执行时，彼此之间不应该存在相互影响。隔离程度不是绝对的，每个数据库都提供有自己的隔离级别，每个数据库的默认隔离级别也不尽相同。

 **持久性（Durability）：**事务正常执行完毕后，对数据库的修改是永久性的。即事务的修改操作已经记录到了存储介质中。

# MySQL的四种隔离级别

**未提交读 Read Uncommitted** 在该隔离级别，一个事务可以读到了另一个未提交事务修改过的数据。本隔离级别很少用于实际应用，因为它的性能也不比其他级别好多少。

> 事务可读到未提交的数据也叫脏读（Dirty Read），由于脏读在实际应用中会导致很多问题，一般这类隔离级别应用很少。

**提交后读 Read Committed** 这是大多数数据库系统的默认隔离级别（但不是MySQL默认的）。它满足了隔离的简单定义：一个事务只能读到另一个已经提交的事务修改过的数据，并且其他事务每对该数据进行一次修改并提交后，该事务都能查询得到最新值。这种隔离级别  也支持所谓的不可重复读（Nonrepeatable  Read），因为同一事务的其他实例在该实例处理其间可能会有新的commit，所以同一select可能返回不同结果。

**可重复读 Repeatable Read** MySQL的默认事务隔离级别。一个事务只能读到另一个已经提交的事务修改过的数据，但是第一次读过某条记录后，即使其他事务修改了该记录的值并且提交，该事务之后再读该条记录时，读到的仍是第一次读到的值，而不是每次都读到不同的数据。但是这会导致另一个问题：**幻读**

> 幻读指当用户读取某一范围的数据行时，另一个事务又在该范围内插入了新行，当用户再读取该范围的数据行时，会发现有新的“幻影”  行。InnoDB和Falcon存储引擎通过多版本并发控制（MVCC，Multiversion Concurrency  Control）机制解决了该问题。

**串行化Serializable** 这是最高的隔离级别，它通过强制事务排序，使之不可能相互冲突，从而解决幻读问题。简言之，它是在每个读的数据行上加上共享锁。在这个级别，可能导致大量的超时现象和锁竞争。

简单总结一下

- **Serializable** 可避免脏读、不可重复读、幻读的发生
- **Repeatable read** 可避免脏读、不可重复读的发生
- **Read committed**  可避免脏读的发生
- **Read uncommitted** 最低级别，任何情况都无法保证。

不可重复读和幻读容易搞混，确实这两者有些相似。但不可重复读重点在于update和delete，而幻读的重点在于insert。

# 测试
准备数据

<pre class="line-numbers"><code class="language-sql">
CREATE TABLE user (
    id INT PRIMARY KEY,
    name VARCHAR(32)
) Engine=InnoDB CHARSET=utf8;
</code></pre>

当前数据库的隔离级别是Repeatable read，事务自动提交
<pre class="line-numbers"><code class="language-sql">
mysql> show variables like '%isolation%';
+-----------------------+-----------------+
| Variable_name         | Value           |
+-----------------------+-----------------+
| transaction_isolation | REPEATABLE-READ |
+-----------------------+-----------------+
1 row in set (0.02 sec)

mysql> show variables like 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | ON    |
+---------------+-------+
1 row in set (0.04 sec)
</code></pre>

## read uncommitted
打开两个session，分别设置为事务隔离级别为**read uncommitted**，事务不自动提交
<pre class="line-numbers"><code class="language-sql">
mysql> set session transaction isolation level read uncommitted;
Query OK, 0 rows affected (0.00 sec)

mysql> show session variables like '%isolation%';
+-----------------------+------------------+
| Variable_name         | Value            |
+-----------------------+------------------+
| transaction_isolation | READ-UNCOMMITTED |
+-----------------------+------------------+
1 row in set (0.02 sec)

mysql> set session autocommit=0;
Query OK, 0 rows affected (0.00 sec)

mysql> show session variables like 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | OFF   |
+---------------+-------+
1 row in set (0.04 sec)

mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)
</code></pre>
然后按下面的顺序操作
sessionA 
<pre class="line-numbers"><code class="language-sql">
mysql> select * from user where name = 'edgar';
Empty set
</code></pre>
sessionB
<pre class="line-numbers"><code class="language-sql">
mysql> insert user(id, name) values(1, 'edgar');
Query OK, 1 row affected (0.03 sec)
mysql> select * from user where name = 'edgar';
+----+-------+
| id | name  |
+----+-------+
|  1 | edgar |
+----+-------+
1 row in set (0.02 sec)
</code></pre>
sessionA 
<pre class="line-numbers"><code class="language-sql">
mysql> select * from user where name = 'edgar';
+----+-------+
| id | name  |
+----+-------+
|  1 | edgar |
+----+-------+
1 row in set (0.03 sec)
</code></pre>
sessionB
<pre class="line-numbers"><code class="language-sql">
mysql> rollback;
Query OK, 0 rows affected (0.06 sec)
</code></pre>
sessionA 
<pre class="line-numbers"><code class="language-sql">
mysql> select * from user where name = 'edgar';
Empty set
</code></pre>

可以看到sessionA读到了不存在的数据

## read committed
打开两个session，分别设置为事务隔离级别为**read committed**，事务不自动提交
<pre class="line-numbers"><code class="language-sql">
mysql> set session transaction isolation level read committed;
Query OK, 0 rows affected (0.00 sec)

mysql> show session variables like '%isolation%';
+-----------------------+----------------+
| Variable_name         | Value          |
+-----------------------+----------------+
| transaction_isolation | READ-COMMITTED |
+-----------------------+----------------+
1 row in set (0.03 sec)

mysql> show session variables like 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | OFF   |
+---------------+-------+
1 row in set (0.04 sec)

mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)
</code></pre>
然后按下面的顺序操作
sessionA 
<pre class="line-numbers"><code class="language-sql">
mysql> select * from user where id = 1;
Empty set
</code></pre>
sessionB
<pre class="line-numbers"><code class="language-sql">
insert user(id, name) values(1, 'edgar');
Query OK, 1 row affected (0.00 sec)

mysql> select * from user where id = 1;
+----+-------+
| id | name  |
+----+-------+
|  1 | edgar |
+----+-------+
1 row in set (0.02 sec)
</code></pre>
sessionA 
<pre class="line-numbers"><code class="language-sql">
mysql> select * from user where id = 1;
Empty set
</code></pre>
sessionB
<pre class="line-numbers"><code class="language-sql">
mysql> commit;
Query OK, 0 rows affected (0.07 sec)
</code></pre>
sessionA 
<pre class="line-numbers"><code class="language-sql">
mysql> select * from user where id = 1;
+----+-------+
| id | name  |
+----+-------+
|  1 | edgar |
+----+-------+
1 row in set (0.03 sec)
</code></pre>
sessionB
<pre class="line-numbers"><code class="language-sql">
mysql> update user set name = 'unkown' where id = 1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
</code></pre>
sessionA 
<pre class="line-numbers"><code class="language-sql">
mysql> select * from user where id = 1;
+----+-------+
| id | name  |
+----+-------+
|  1 | edgar |
+----+-------+
1 row in set (0.03 sec)
</code></pre>
sessionB
<pre class="line-numbers"><code class="language-sql">
mysql> delete from user where id = 1;
Query OK, 1 row affected (0.00 sec)
</code></pre>
sessionA 
<pre class="line-numbers"><code class="language-sql">
mysql> select * from user where id = 1;
+----+-------+
| id | name  |
+----+-------+
|  1 | edgar |
+----+-------+
1 row in set (0.03 sec)
</code></pre>
sessionB
<pre class="line-numbers"><code class="language-sql">
mysql> commit;
Query OK, 0 rows affected (0.01 sec)
</code></pre>
sessionA 
<pre class="line-numbers"><code class="language-sql">
mysql> select * from user where id = 1;
Empty set
</code></pre>

## repeatable read
打开两个session，分别设置为事务隔离级别为** repeatable read**，事务不自动提交
<pre class="line-numbers"><code class="language-sql">
mysql> set session transaction isolation level  repeatable read;
Query OK, 0 rows affected (0.00 sec)

mysql> show session variables like '%isolation%';
+-----------------------+-----------------+
| Variable_name         | Value           |
+-----------------------+-----------------+
| transaction_isolation | REPEATABLE-READ |
+-----------------------+-----------------+
1 row in set (0.01 sec)

mysql>  set session autocommit=0;
Query OK, 0 rows affected (0.00 sec)

mysql> show session variables like 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | OFF   |
+---------------+-------+
1 row in set (0.01 sec)
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)
</code></pre>
然后按下面的顺序操作
sessionA 
<pre class="line-numbers"><code class="language-sql">
mysql> select * from user where id = 1;
Empty set
</code></pre>
sessionB
<pre class="line-numbers"><code class="language-sql">
insert user(id, name) values(1, 'edgar');
Query OK, 1 row affected (0.00 sec)
mysql> select * from user where id = 1;
+----+-------+
| id | name  |
+----+-------+
|  1 | edgar |
+----+-------+
1 row in set (0.02 sec)
</code></pre>
sessionA 
<pre class="line-numbers"><code class="language-sql">
mysql> select * from user where id = 1;
Empty set
</code></pre>
sessionB
<pre class="line-numbers"><code class="language-sql">
mysql> commit;
Query OK, 0 rows affected (0.07 sec)
</code></pre>
sessionA 
<pre class="line-numbers"><code class="language-sql">
mysql> select * from user where id = 1;
Empty set

mysql> commit;
Query OK, 0 rows affected (0.00 sec)

mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where id > 1;
Empty set

</code></pre>
sessionB
<pre class="line-numbers"><code class="language-sql">
mysql> insert user(id, name) values(2, 'jennifer');
Query OK, 1 row affected (0.00 sec)
</code></pre>
sessionA 
<pre class="line-numbers"><code class="language-sql">
mysql> select * from user where id > 1;
Empty set

mysql> insert user(id, name) values(2, 'jennifer');
-- 卡住，当sessionB commit的时候返回如下信息
1062 - Duplicate entry '2' for key 'PRIMARY'
</code></pre>
通过上面的操作，我们已经看到sessionB对sessionA造成的**幻读**影响：sessionA看到大于1的数据为空集，insert的时候却返回主键冲突

## Serializable
打开两个session，分别设置为事务隔离级别为**Serializable**，事务不自动提交
<pre class="line-numbers"><code class="language-sql">
mysql> set session transaction isolation level Serializable;
Query OK, 0 rows affected (0.00 sec)

mysql> show session variables like '%isolation%';
+-----------------------+--------------+
| Variable_name         | Value        |
+-----------------------+--------------+
| transaction_isolation | SERIALIZABLE |
+-----------------------+--------------+
1 row in set (0.01 sec)

mysql> set session autocommit=0;
Query OK, 0 rows affected (0.00 sec)

mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)
</code></pre>
然后按下面的顺序操作
sessionA 
<pre class="line-numbers"><code class="language-sql">
mysql> update user set name = 'leona' where id = 1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
</code></pre>
sessionB
<pre class="line-numbers"><code class="language-sql">
mysql> select * from user where id = 1;
-- 会卡住没有返回，
</code></pre>
sessionA 
<pre class="line-numbers"><code class="language-sql">
mysql> commit;
Query OK, 0 rows affected (0.01 sec)
</code></pre>
sessionB返回
<pre class="line-numbers"><code class="language-sql">
+----+-------+
| id | name  |
+----+-------+
|  1 | edgar |
+----+-------+
1 row in set (46.59 sec)
</code></pre>
如果sessionA一直没有提交，sessionB会超时
<pre class="line-numbers"><code class="language-sql">
1205 - Lock wait timeout exceeded; try restarting transaction
</code></pre>
# 幻读

不可重复读和幻读容易搞混，确实这两者有些相似。但不可重复读重点在于update和delete，而幻读的重点在于insert。

**幻读错误的理解：**很多地方说幻读是 事务A 执行两次 select 操作得到不同的数据集，即 select 1 得到 10 条记录，select 2 得到 11 条记录。这其实并不是幻读，这是不可重复读的一种，在read uncommitted和read committed下会出现。但在repeatable read下不会出现

mysql 的幻读并不是说两次读取获取的结果集不同，幻读侧重的方面是某一次的 select 操作得到的结果所表征的数据状态无法支撑后续的业务操作。具体来说而是事务在插入事先检测不存在的记录时，惊奇的发现这些数据已经存在了，之前的检测读获取到的数据如同鬼影一般。

>  select 某记录是否存在，不存在，准备插入此记录，但执行 insert 时发现此记录已存在，无法插入，此时就发生了幻读。

> 知乎上的列子
>
> https://www.zhihu.com/question/47007926

users： id 主键

1. T1：select * from users where id = 1;

2. T2：insert into `users`(`id`, `name`) values (1, 'big cat');

3. T1：insert into `users`(`id`, `name`) values (1, 'big cat');

T1 ：主事务，检测表中是否有 id 为 1 的记录，没有则插入，这是我们期望的正常业务逻辑。

T2 ：干扰事务，目的在于扰乱 T1 的正常的事务执行。

在repeatable read下1、2 是会正常执行的，3 则会报错主键冲突，对于 T1 的业务来说是执行失败的，这里 T1 就是发生了幻读，因为T1读取的数据状态并不能支持他的下一步的业务，见鬼了一样：“我刚才读到的结果应该可以支持我这样操作才对啊，为什么现在不可以”。

如果T1不相信又执行一次`select * from users where id = 1;`依然查询不到id=1的数据，此时，幻读无疑已经发生，T1 无论读取多少次，都查不到 id = 1 的记录，但它的确无法插入这条他通过读取来认定不存在的记录（此数据已被T2插入），对于 T1 来说，它幻读了。

# 为什么mysql选可重复读作为默认的隔离级别

这是历史原因造成的。binlog有三种格式

- statement:记录的是修改SQL语句
- row：记录的是每行实际数据的变更
- mixed：statement和row模式的混合 

那Mysql在5.0这个版本以前，binlog只支持`STATEMENT`这种格式！而这种格式在**读已提交(Read Committed)**这个隔离级别下主从复制是有bug的，因此Mysql将**可重复读(Repeatable Read)**作为默认的隔离级别！

举个例子主从模式下：

```
mysql> select * from user where id = 1;
+----+-------+
| id | name  |
+----+-------+
|  1 | edgar |
+----+-------+
1 row in set (0.59 sec)
```
在read committed下执行下列语句

1. T1：`begin;`
2. T1：`delete from user where id < 10;`
3. T2：`begin;`
4. T2：`insert user values(5,'Edgar');`
4. T2：`commit;`
5. T1：`commit;`

就会出现主从不一致性的问题！原因其实很简单，就是在master上执行的顺序为先删后插！而此时binlog为STATEMENT格式，它记录的顺序为先插后删！从(slave)同步的是binglog，因此从机执行的顺序和主机不一致！就会出现主从不一致！

如果使用repeated read隔离级别，`delete from user where id < 10;`会加上临界锁（行锁+间隙锁）,`insert user values(5,'Edgar');`会被阻塞。

如果将binglog的格式修改为row格式，此时是基于行的复制，自然就不会出现sql执行顺序不一样的问题。但是这个格式在mysql5.1版本开始才引入。因此由于历史原因，mysql将默认的隔离级别设为可重复读(Repeatable Read)，保证主从复制不出问题！

**互联网项目为什么选read committed隔离级别**

> 网上看的，没有考证

- 在RR隔离级别下，存在间隙锁，导致出现死锁的几率比RC大的多
- 在RR隔离级别下，条件列未命中索引会锁表！而在RC隔离级别下，只锁行。

> 在RC隔离级别下，其先走聚簇索引，进行全部扫描。但在实际中，MySQL做了优化，在MySQL Server过滤条件，发现不满足后，会调用unlock_row方法，把不满足条件的记录释放锁。在RR隔离级别下，会使用间隙锁将表都锁上

- 在RC隔离级别下，半一致性读(semi-consistent)特性增加了update操作的并发性！

在5.1.15的时候，innodb引入了一个概念叫做“semi-consistent”，减少了更新同一行记录时的冲突，减少锁等待。
所谓半一致性读就是，一个update语句，如果读到一行已经加锁的记录，此时InnoDB返回记录最近提交的版本，由MySQL上层判断此版本是否满足update的where条件。若满足(需要更新)，则MySQL会重新发起一次读操作，此时会读取行的最新版本(并加锁)！
具体表现如下:
此时有两个Session，Session1和Session2！
Session1执行

```
update test set color = 'blue' where color = 'red'; 
```

先不Commit事务！
与此同时Ssession2执行

```
update test set color = 'blue' where color = 'white'; 
```

session  2尝试加锁的时候，发现行上已经存在锁，InnoDB会开启semi-consistent  read，返回最新的committed版本(1,red),(2，white),(5,red),(7,white)。MySQL会重新发起一次读操作，此时会读取行的最新版本(并加锁)!
而在RR隔离级别下，Session2只能等待！


# 参考资料
https://mp.weixin.qq.com/s/Jeg8656gGtkPteYWrG5_Nw

https://www.cnblogs.com/csniper/p/5525477.html

https://mp.weixin.qq.com/s/gkfzOtYWWhMgMgcRB32BoA

https://mp.weixin.qq.com/s/x_7E2R2i27Ci5O7kLQF0UA
