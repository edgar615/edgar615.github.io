---
layout: post
title: MySQL锁（2）-死锁
date: 2018-04-26
categories:
    - MySQL
comments: true
permalink: innodb-dead-lock.html
---


# 1. 死锁
**死锁产生**

- 死锁是指两个或多个事务在同一资源上相互占用，并请求锁定对方占用的资源，从而导致恶性循环
- 当事务试图以不同的顺序锁定资源时，就可能产生死锁。多个事务同时锁定同一个资源时也可能会产生死锁
-  锁的行为和顺序和存储引擎相关。以同样的顺序执行语句，有些存储引擎会产生死锁有些不会——死锁有双重原因：真正的数据冲突；存储引擎的实现方式。

**检测死锁**

数据库系统实现了各种死锁检测和死锁超时的机制。InnoDB存储引擎能检测到死锁的循环依赖并立即返回一个错误。

**死锁恢复**

死锁发生以后，只有部分或完全回滚其中一个事务，才能打破死锁，InnoDB目前处理死锁的方法是，将持有最少行级排他锁的事务进行回滚。所以事务型应用程序在设计时必须考虑如何处理死锁，多数情况下只需要重新执行因死锁回滚的事务即可。

**外部锁的死锁检测**

发生死锁后，InnoDB 一般都能自动检测到，并使一个事务释放锁并回退，另一个事务获得锁，继续完成事务。但在涉及外部锁，或涉及表锁的情况下，InnoDB 并不能完全自动检测到死锁， 这需要通过设置锁等待超时参数`innodb_lock_wait_timeout`来解决 

**死锁影响性能**

死锁会影响性能而不是会产生严重错误，因为InnoDB会自动检测死锁状况并回滚其中一个受影响的事务。在高并发系统上，当许多线程等待同一个锁时，死锁检测可能导致速度变慢。 有时当发生死锁时，禁用死锁检测（使用`innodb_deadlock_detect`配置选项）可能会更有效，这时可以依赖`innodb_lock_wait_timeout`设置进行事务回滚。 

# 2. 查看死锁日志

> 直接从下面这篇文章中复制
>
> https://mp.weixin.qq.com/s/9H4e2c2iqB1Iqz4npXGrjA

我们通过`show engine innodb status` 查看的日志是最新一次记录死锁的日志。

```
        ------------------------

        LATEST DETECTED DEADLOCK

        ------------------------

        2017-09-09 22:34:13 7f78eab82700

        *** (1) TRANSACTION: #事务1

        TRANSACTION 462308399, ACTIVE 33 sec starting index read 

        mysql tables in use 1, locked 1

        LOCK WAIT 2 lock struct(s), heap size 360, 1 row lock(s)

        MySQL thread id 3525577, OS thread handle 0x7f896cc4b700, query id 780039657 localhost root updating

        delete from ty where a=5

        *** (1) WAITING FOR THIS LOCK TO BE GRANTED:

        RECORD LOCKS space id 219 page no 4 n bits 72 index `idxa` of table `test`.`ty` trx id 462308399 lock_mode X waiting

        *** (2) TRANSACTION:

        TRANSACTION 462308398, ACTIVE 61 sec inserting, thread declared inside InnoDB 5000

        mysql tables in use 1, locked 1

        5 lock struct(s), heap size 1184, 4 row lock(s), undo log entries 2

        MySQL thread id 3525490, OS thread handle 0x7f78eab82700, query id 780039714 localhost root update

        insert into ty(a,b) values(2,10)

        *** (2) HOLDS THE LOCK(S):

        RECORD LOCKS space id 219 page no 4 n bits 72 index `idxa` of table `test`.`ty` trx id 462308398 lock_mode X

        *** (2) WAITING FOR THIS LOCK TO BE GRANTED:

        RECORD LOCKS space id 219 page no 4 n bits 72 index `idxa` of table `test`.`ty` trx id 462308398 lock_mode X locks gap before rec insert intention waiting

        *** WE ROLL BACK TRANSACTION (1)
```

** (1) TRANSACTION: #事务1**
TRANSACTION 462308399, ACTIVE 33 sec starting index read** 
事务编号为 462308399 ，活跃33秒，starting index read 表示事务状态为根据索引读取数据。常见的其他状态:

> 1. fetching rows 表示事务状态在row_search_for_mysql中被设置，表示正在查找记录。
> 2. updating or deleting 表示事务已经真正进入了Update/delete的函数逻辑（row_update_for_mysql）
> 3. thread declared inside InnoDB 说明事务已经进入innodb层。通常而言 不在innodb层的事务大部分是会被回滚的。

**mysql tables in use 1**, 说明当前的事务使用一个表。locked 1 表示表上有一个表锁,对于DML语句为LOCK_IX
**LOCK WAIT 2 lock struct(s), heap size 360, 1 row lock(s)**
**LOCK WAIT**表示正在等待锁, 2 lock struct(s) 表示trx->trx_locks锁链表的长度为2，每个链表节点代表该事务持有的一个锁结构，包括表锁，记录锁以及auto_inc锁等。本案例中**2locks** 表示**IX**锁和 **lock_mode X (Next-key lock)**
**heap size 360** 表示事务分配的锁堆内存大小,一般没有什么具体的用处。
**1 row lock(s)**表示当前事务持有的行记录锁/gap 锁的个数。
**delete from ty where a=5** 表示事务1在执行的sql ，**不过比较悲伤的事情是show engine innodb status 是查看不到完整的事务的sql 的，通常显示当前正在等待锁的sql。**

 **(1) WAITING FOR THIS LOCK TO BE GRANTED:**

**RECORD LOCKS space id 219 page no 4 n bits 72 index `idxa` of table `test`.`ty` trx id 462308399 lock_mode X waiting**

**RECORD LOCKS** 表示记录锁,space id为219,page号4 ，n bits 72表示这个聚集索引记录锁结构上留有72个Bit位

表示事务1 正在等待表 ty 上的 idxa 的 X 锁本案例中其实是Next-Key lock

事务2的log 和上面分析类似，

**(2) HOLDS THE LOCK(S):**

**RECORD LOCKS space id 219 page no 4 n bits 72 index `idxa` of table `test`.`ty` trx id 462308398 lock_mode X**

显示了事务2 insert into ty(a,b) values(2,10)持有了a=5 的Lock mode X |LOCK_GAP ，**不过我们从日志里面看不到 事务2 执行的 delete from  ty where  a=5;这点也是造成DBA 仅仅根据日志难以分析死锁的问题的根本原因。**

(2) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 219 page no 4 n bits 72 index `idxa` of table  `test`.`ty` trx id 462308398 lock_mode X locks gap before rec insert  intention waiting

表示事务2的insert 语句正在等待插入意向锁 lock_mode X locks gap before rec insert intention waiting (LOCK_X + LOCK_REC_GAP )

这里需要各位注意的是锁组合，类似**lock_mode X waiting ,lock_mode X,lock_mode X locks gap before rec insert intention waiting** 是我们分析死锁的核心重点。如何理解锁组合呢？

首先我们要知道对于MySQL有两种常规锁模式

- LOCK_S（读锁，共享锁）

- LOCK_X（写锁，排它锁）

最容易理解的锁模式，读加共享锁，写加排它锁.

有如下几种锁的属性

- **LOCK_REC_NOT_GAP     （锁记录）**

- **LOCK_GAP             （锁记录前的GAP）**

- **LOCK_ORDINARY        （同时锁记录+记录前的GAP 。传说中的Next Key锁）**

- **LOCK_INSERT_INTENTION（插入意向锁，其实是特殊的GAP锁）**

锁的属性可以与锁模式任意组合。例如.

- **lock->type_mode    可以是Lock_X 或者Lock_S** 

- **locks gap before rec  表示为gap锁：lock->type_mode & LOCK_GAP**

- **locks rec but not gap 表示为记录锁，非gap锁：lock->type_mode & LOCK_REC_NOT_GAP**

- **insert intention      表示为插入意向锁：lock->type_mode & LOCK_INSERT_INTENTION**

- **waiting            表示锁等待：lock->type_mode & LOCK_WAIT**

**Innodb 日志中如果提示 lock mode S/lock mode X ，其实都是 gap 锁，如果是行记录锁会提示 but not gap**

# 2. 对同一字段加锁引发的死锁

## 2.1. 普通索引

数据准备

```
CREATE TABLE `ty` (
	`id` INT ( 11 ) NOT NULL AUTO_INCREMENT,
	`a` INT ( 11 ) DEFAULT NULL,
	`b` INT ( 11 ) DEFAULT NULL,
	PRIMARY KEY ( `id` ),
	KEY `idx_a` ( `a` ) 
);
insert into ty(a,b) values(2,3),(5,4),(6,7);
```

事务1

```
mysql> delete from ty where a = 5;
Query OK, 1 row affected (0.00 sec)
```

事务2

```
mysql> delete from ty where a = 5;
// 等待锁
```

事务1

```
mysql> insert into ty(a, b) values(2, 10);
Query OK, 1 row affected (0.00 sec)
```

事务2

```
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

通过`show engine innodb status `查看日志

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2020-10-10 10:30:04 0x7f41974f8700
*** (1) TRANSACTION:
TRANSACTION 1379105, ACTIVE 15 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 1375190, OS thread handle 139919972861696, query id 4125557 localhost root updating
delete from ty where a = 5

*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 46 page no 5 n bits 72 index idx_a of table `test`.`ty` trx id 1379105 lock_mode X waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 2; compact format; info bits 32
 0: len 4; hex 80000005; asc     ;;
 1: len 4; hex 80000002; asc     ;;


*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 46 page no 5 n bits 72 index idx_a of table `test`.`ty` trx id 1379105 lock_mode X waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 2; compact format; info bits 32
 0: len 4; hex 80000005; asc     ;;
 1: len 4; hex 80000002; asc     ;;


*** (2) TRANSACTION:
TRANSACTION 1379104, ACTIVE 34 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 5 lock struct(s), heap size 1136, 4 row lock(s), undo log entries 2
MySQL thread id 1375191, OS thread handle 139919972570880, query id 4125558 localhost root update
insert into ty(a, b) values(2, 10)

*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 46 page no 5 n bits 72 index idx_a of table `test`.`ty` trx id 1379104 lock_mode X
Record lock, heap no 3 PHYSICAL RECORD: n_fields 2; compact format; info bits 32
 0: len 4; hex 80000005; asc     ;;
 1: len 4; hex 80000002; asc     ;;


*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 46 page no 5 n bits 72 index idx_a of table `test`.`ty` trx id 1379104 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 2; compact format; info bits 32
 0: len 4; hex 80000005; asc     ;;
 1: len 4; hex 80000002; asc     ;;

*** WE ROLL BACK TRANSACTION (1)
```

首先要理解的是 **对同一个字段申请加锁是需要排队。**

其次表 ty 中索引 idxa 为非唯一普通索引，我们根据事务执行的时间顺序来解释。

1. 根据死锁日志显示事务 1 也即`1379104`执行的事务，根据 HOLDS THE LOCK(S) 显示 事务1先执行 delete from ty where a=5 ，该事务持有索引 a=5 的行锁 lock_mode X ,因为是 RR 隔离级别，所以 事务1 还持有两个 gap 锁[1,2]-[2,5], [2,5]-[3,6] 。
2. 事务2的日志也即 `1379105` 执行的事务,申请对 a=5 加锁，一个 rec lock 和两个 gap 锁,因为 事务1 中 delete 还没释放，事务 2 等待事务 1释放 a=5的锁资源。
3. 根据` WAITING FOR THIS LOCK TO BE GRANTED;`事务1的insert正在等待`lock_mode X locks gap before rec insert intention waiting`，因为 insert 语句 介于 gap 锁[1,2]-[2,5]之间。insert 语句必须等待前面 事务2 中 delete 获取锁并且释放锁。于是，事务2(delete) 等待事务1(delete) ,事务1(insert)等待 事务2(delete),循环等待，造成死锁。



可以通过查询`data_locks`表验证

```
mysql> select * from data_locks;
+--------+----------------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+------------------------+-------------+-----------+
| ENGINE | ENGINE_LOCK_ID                         | ENGINE_TRANSACTION_ID | THREAD_ID | EVENT_ID | OBJECT_SCHEMA | OBJECT_NAME | PARTITION_NAME | SUBPARTITION_NAME | INDEX_NAME | OBJECT_INSTANCE_BEGIN | LOCK_TYPE | LOCK_MODE              | LOCK_STATUS | LOCK_DATA |
+--------+----------------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+------------------------+-------------+-----------+
| INNODB | 139920103510256:1105:139920035367776   |               1379104 |   1375328 |       22 | test          | ty          | NULL           | NULL              | NULL       |       139920035367776 | TABLE     | IX                     | GRANTED     | NULL      |
| INNODB | 139920103510256:46:5:3:139920035364672 |               1379104 |   1375328 |       22 | test          | ty          | NULL           | NULL              | idx_a      |       139920035364672 | RECORD    | X                      | GRANTED     | 5, 2      |
| INNODB | 139920103510256:46:4:3:139920035365016 |               1379104 |   1375328 |       22 | test          | ty          | NULL           | NULL              | PRIMARY    |       139920035365016 | RECORD    | X,REC_NOT_GAP          | GRANTED     | 2         |
| INNODB | 139920103510256:46:5:4:139920035365360 |               1379104 |   1375328 |       22 | test          | ty          | NULL           | NULL              | idx_a      |       139920035365360 | RECORD    | X,GAP                  | GRANTED     | 6, 3      |
| INNODB | 139920103510256:46:5:5:139920035365360 |               1379104 |   1375328 |       22 | test          | ty          | NULL           | NULL              | idx_a      |       139920035365360 | RECORD    | X,GAP                  | GRANTED     | 2, 4      |
| INNODB | 139920103510256:46:5:3:139920035365704 |               1379104 |   1375328 |       23 | test          | ty          | NULL           | NULL              | idx_a      |       139920035365704 | RECORD    | X,GAP,INSERT_INTENTION | GRANTED     | 5, 2      |
+--------+----------------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+------------------------+-------------+-----------+
6 rows in set (0.00 sec)

```

## 2.2. 唯一索引

```
CREATE TABLE `ty` (
	`id` INT ( 11 ) NOT NULL AUTO_INCREMENT,
	`a` INT ( 11 ) DEFAULT NULL,
	`b` INT ( 11 ) DEFAULT NULL,
	PRIMARY KEY ( `id` ),
	UNIQUE KEY `uq_a` ( `a` ) 
);
insert into ty(a,b) values(2,3),(5,4),(6,7);
```

事务1

```
mysql> delete from ty where a = 5;
Query OK, 1 row affected (0.00 sec)
```

事务2

```
mysql> delete from ty where a = 5;
// 等待锁
```

事务1

```
mysql> insert into ty(a, b) values(3, 10);
Query OK, 1 row affected (0.00 sec)
```

事务2

```
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

通过`show engine innodb status `查看日志，发现事务1持有的锁类型发生了变化` lock_mode X locks rec but not gap Record lock`，之前是`lock_mode X Record lock`。 原因是因为测试表结构发生了变化字段 a 由普通索引变为唯一键，RR 模式下对唯一键操作是没有 gap 锁的，而且 insert 写入含有唯一键的数据是会申请 GAP 锁的特殊情况 Insert Intention Lock。

1. 根据 HOLDS THE LOCK(S)显示 事务1 先执行 `delete from ty where a=5` ，该事务持有索引 a=5 的行锁 `lock_mode X locks rec but not gap`。因为本例中 a 是唯一键，故没有 gap 锁。

2. 事务 2申请对 a=5 加锁(X Next-key Lock)，一个 rec lock 但是因为 事务1 中 delete 已经执行完成，记录无效没有被删除，锁还没释放，故 事务1等待事务1 释放 a=5 的锁资源，日志中提示 lock_mode X waiting。

3. 然后根据 `lock_mode X locks gap before rec insert intention waiting`，提示事务 1 insert 语句正在等待事务2 中 delete 获取锁并且释放锁。因为 a 字段是一个唯一索引，所以 insert 语句会在插入前进行一次 duplicate key 的检查，需要申请 S 锁防止其他事务对 a 字段进行重复插入。**于是事务2(delete) 等待事务1(delete) ,事务1(insert)等待事务2(delete),循环等待，造成死锁**。

# 3. 唯一索引冲突引发的死锁

并发条件下，唯一键索引冲突可能会导致死锁，这种死锁一般分为两种，一种是`rollback`引发，另一种是`commit`引发。

## 3.1. 两个事务并发 insert 唯一键冲突和 gap 锁引发的死锁

准备数据

```
create table t7(

	id int not null primary key auto_increment,

	a int not null ,

	unique key ua(a)

) engine=innodb;

insert into t7(id,a) values(1,1),(5,4),(20,20),(25,12);
```

T1

```
mysql> insert into t7(id, a) values(26, 10);
Query OK, 1 row affected (0.00 sec)
```

T2

```
mysql> insert into t7(id, a) values(30, 10);
// 等待
```

T1

```
mysql> insert into t7(id, a) values(40, 9);
Query OK, 1 row affected (0.00 sec)
```

T2

```
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

查看锁

```
mysql> select * from data_locks;
+--------+----------------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+---------------+-------------+-----------+
| ENGINE | ENGINE_LOCK_ID                         | ENGINE_TRANSACTION_ID | THREAD_ID | EVENT_ID | OBJECT_SCHEMA | OBJECT_NAME | PARTITION_NAME | SUBPARTITION_NAME | INDEX_NAME | OBJECT_INSTANCE_BEGIN | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA |
+--------+----------------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+---------------+-------------+-----------+
| INNODB | 139920103510256:1111:139920035367776   |               1379303 |   1375328 |      102 | test          | t7          | NULL           | NULL              | NULL       |       139920035367776 | TABLE     | IX            | GRANTED     | NULL      |
| INNODB | 139920103510256:52:5:6:139920035364672 |               1379303 |   1375328 |      102 | test          | t7          | NULL           | NULL              | ua         |       139920035364672 | RECORD    | S             | WAITING     | 10, 26    |
| INNODB | 139920103509400:1111:139920035361616   |               1379298 |   1375327 |      110 | test          | t7          | NULL           | NULL              | NULL       |       139920035361616 | TABLE     | IX            | GRANTED     | NULL      |
| INNODB | 139920103509400:52:5:6:139920035358512 |               1379298 |   1375328 |      102 | test          | t7          | NULL           | NULL              | ua         |       139920035358512 | RECORD    | X,REC_NOT_GAP | GRANTED     | 10, 26    |
+--------+----------------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+---------------+-------------+-----------+
4 rows in set (0.00 sec)

```

T1执行insert成功，持有a=1的X行锁（X,REC_NOT_GAP）

T2执行insert，因为 T1 的insert 已经插入 a=10 的记录，T2的 insert a=10 则发生唯一约束冲突，需要申请对冲突的唯一索引加上 S Next-key Lock (也即是 lock mode S waiting ) 这是一个间隙锁会申请锁住(,10],(10,20]之间的 gap 区域。从这里会发现，即使是 RC 事务隔离级别，也同样会存在 Next-Key Lock 锁，从而阻塞并发。

事务T2 insert into t7(id,a) values(40,9) 该语句插入的 a=9 的值在事务 T1 申请的 gap  锁[4,10]之间，故需事务 T2 的第二条 insert 语句要等待事务 T1 的 S-Next-key Lock 锁释放，在日志中显示  lock_mode X locks gap before rec insert intention waiting。

## 3.1.  `rollback`引发的死锁

开启3个事务，都执行下面的insert

```
INSERT INTO user (id, username, nickname, gender, age) VALUES (10, 'username10', 'nickname10', 2, 10);
```

可以看到3个事务都需要对记录10加锁，不同的事务1加的X锁，事务2，事务3加的S锁（此时事务2和事务3插入数据时检测到了重复键错误，事务2和事务3要在这条索引记录上设置S锁，由于X锁的存在，S锁的获取被阻塞。）

```
+------+---------------------------------------+---------------------+---------+--------+-------------+-----------+--------------+-----------------+----------+---------------------+---------+-------------+-----------+---------+
|ENGINE|ENGINE_LOCK_ID                         |ENGINE_TRANSACTION_ID|THREAD_ID|EVENT_ID|OBJECT_SCHEMA|OBJECT_NAME|PARTITION_NAME|SUBPARTITION_NAME|INDEX_NAME|OBJECT_INSTANCE_BEGIN|LOCK_TYPE|LOCK_MODE    |LOCK_STATUS|LOCK_DATA|
+------+---------------------------------------+---------------------+---------+--------+-------------+-----------+--------------+-----------------+----------+---------------------+---------+-------------+-----------+---------+
|INNODB|140622813693800:1174:140622732294672   |25160586             |108      |14      |ed_test      |user       |NULL          |NULL             |NULL      |140622732294672      |TABLE    |IX           |GRANTED    |NULL     |
|INNODB|140622813693800:117:4:6:140622732291568|25160586             |108      |14      |ed_test      |user       |NULL          |NULL             |PRIMARY   |140622732291568      |RECORD   |S,REC_NOT_GAP|WAITING    |10       |
|INNODB|140622813692952:1174:140622732288576   |25160585             |107      |21      |ed_test      |user       |NULL          |NULL             |NULL      |140622732288576      |TABLE    |IX           |GRANTED    |NULL     |
|INNODB|140622813692952:117:4:6:140622732285472|25160585             |107      |21      |ed_test      |user       |NULL          |NULL             |PRIMARY   |140622732285472      |RECORD   |S,REC_NOT_GAP|WAITING    |10       |
|INNODB|140622813691256:1174:140622732276320   |25160584             |104      |35      |ed_test      |user       |NULL          |NULL             |NULL      |140622732276320      |TABLE    |IX           |GRANTED    |NULL     |
|INNODB|140622813691256:117:4:6:140622732273216|25160584             |107      |21      |ed_test      |user       |NULL          |NULL             |PRIMARY   |140622732273216      |RECORD   |X,REC_NOT_GAP|GRANTED    |10       |
+------+---------------------------------------+---------------------+---------+--------+-------------+-----------+--------------+-----------------+----------+---------------------+---------+-------------+-----------+---------+


+------+---------------------------------------+--------------------------------+--------------------+-------------------+--------------------------------+---------------------------------------+------------------------------+------------------+-----------------+------------------------------+
|ENGINE|REQUESTING_ENGINE_LOCK_ID              |REQUESTING_ENGINE_TRANSACTION_ID|REQUESTING_THREAD_ID|REQUESTING_EVENT_ID|REQUESTING_OBJECT_INSTANCE_BEGIN|BLOCKING_ENGINE_LOCK_ID                |BLOCKING_ENGINE_TRANSACTION_ID|BLOCKING_THREAD_ID|BLOCKING_EVENT_ID|BLOCKING_OBJECT_INSTANCE_BEGIN|
+------+---------------------------------------+--------------------------------+--------------------+-------------------+--------------------------------+---------------------------------------+------------------------------+------------------+-----------------+------------------------------+
|INNODB|140622813693800:117:4:6:140622732291568|25160586                        |108                 |14                 |140622732291568                 |140622813691256:117:4:6:140622732273216|25160584                      |107               |21               |140622732273216               |
|INNODB|140622813692952:117:4:6:140622732285472|25160585                        |107                 |21                 |140622732285472                 |140622813691256:117:4:6:140622732273216|25160584                      |107               |21               |140622732273216               |
+------+---------------------------------------+--------------------------------+--------------------+-------------------+--------------------------------+---------------------------------------+------------------------------+------------------+-----------------+------------------------------+

```

此时对事务1执行`rollback`，事务2和事务3发生了死锁。

事务1回滚，由于S锁和S锁是可以兼容的，因此事务2和事务3都获得了这条记录的S锁，此时其中一个事务希望插入，则该事务期望在这条记录上加上X锁，然而另一个事务持有S锁，S锁和X锁互相是不兼容的，两个事务就开始互相等待对方的锁释放，造成了死锁。

**事务2和事务3为什么会加S锁，而不是直接等待X锁**

事务1的insert语句加的是隐式锁(隐式的Record锁、X锁)，但是其他事务插入同一行记录时，出现了唯一键冲突，事务一的隐式锁升级为显示锁。
事务2和事务3在插入之前判断到了唯一键冲突，是因为插入前的**重复索引检查**，这次检查必须进行一次**当前读**，**于是非唯一索引就会被加上S模式的next-key锁，唯一索引就被加上了S模式的Record锁。**
因为插入和更新之前都要进行重复索引检查而执行当前读操作，所以RR隔离级别下，同一个事务内不连续的查询，可能也会出现幻读的效果(但个人并不认为RR 级别下也会出现幻读，幻读的定义应该是连续的读取)。而连续的查询由于都是读取快照，中间没有当前读的操作，所以不会出现幻读。

## 2.2. `commit`引发的死锁

事务1执行删除

```
delete from user where id = 1;
```

事务2和事务3执行新增相同的ID

```
INSERT INTO user (id, username, nickname, gender, age) VALUES (1, 'username10', 'nickname10', 2, 10);
```

事务1执行commit的时候，事务2和事务3发生了死锁。原因和上面相同

# 3. 先insert在update引发的死锁

## 3.1. update非索引字段

一个事务内：insert记录后根据字段p（非索引字段）来update这条记录，然而当出现并发操作的时候，update处会发生dead lock问题

事务1

```

mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> INSERT INTO user (id, username, nickname, gender, age) VALUES (10, 'username10', 'nickname10', 2, 10);
Query OK, 1 row affected (0.01 sec)

```

事务2

```

mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> INSERT INTO user (id, username, nickname, gender, age) VALUES (11, 'username11', 'nickname11', 2, 11);
Query OK, 1 row affected (0.01 sec)

```

事务1

```
mysql> update user set gender = 1 where nickname = 'nickname10';
// 等待锁
```

事务2

```
mysql> update user set gender = 1 where nickname = 'nickname11';
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

查看锁

```
+------+---------------------------------------+---------------------+---------+--------+-------------+-----------+--------------+-----------------+----------+---------------------+---------+-------------+-----------+---------+
|ENGINE|ENGINE_LOCK_ID                         |ENGINE_TRANSACTION_ID|THREAD_ID|EVENT_ID|OBJECT_SCHEMA|OBJECT_NAME|PARTITION_NAME|SUBPARTITION_NAME|INDEX_NAME|OBJECT_INSTANCE_BEGIN|LOCK_TYPE|LOCK_MODE    |LOCK_STATUS|LOCK_DATA|
+------+---------------------------------------+---------------------+---------+--------+-------------+-----------+--------------+-----------------+----------+---------------------+---------+-------------+-----------+---------+
|INNODB|140622813692952:1175:140622732288576   |25160670             |107      |59      |ed_test      |user       |NULL          |NULL             |NULL      |140622732288576      |TABLE    |IX           |GRANTED    |NULL     |
|INNODB|140622813692952:118:4:6:140622732285472|25160670             |104      |94      |ed_test      |user       |NULL          |NULL             |PRIMARY   |140622732285472      |RECORD   |X,REC_NOT_GAP|GRANTED    |11       |
|INNODB|140622813691256:1175:140622732276320   |25160669             |104      |93      |ed_test      |user       |NULL          |NULL             |NULL      |140622732276320      |TABLE    |IX           |GRANTED    |NULL     |
|INNODB|140622813691256:118:4:2:140622732273216|25160669             |104      |94      |ed_test      |user       |NULL          |NULL             |PRIMARY   |140622732273216      |RECORD   |X            |GRANTED    |1        |
|INNODB|140622813691256:118:4:3:140622732273216|25160669             |104      |94      |ed_test      |user       |NULL          |NULL             |PRIMARY   |140622732273216      |RECORD   |X            |GRANTED    |3        |
|INNODB|140622813691256:118:4:4:140622732273216|25160669             |104      |94      |ed_test      |user       |NULL          |NULL             |PRIMARY   |140622732273216      |RECORD   |X            |GRANTED    |5        |
|INNODB|140622813691256:118:4:5:140622732273216|25160669             |104      |94      |ed_test      |user       |NULL          |NULL             |PRIMARY   |140622732273216      |RECORD   |X            |GRANTED    |7        |
|INNODB|140622813691256:118:4:7:140622732273560|25160669             |104      |94      |ed_test      |user       |NULL          |NULL             |PRIMARY   |140622732273560      |RECORD   |X,REC_NOT_GAP|GRANTED    |10       |
|INNODB|140622813691256:118:4:7:140622732273904|25160669             |104      |94      |ed_test      |user       |NULL          |NULL             |PRIMARY   |140622732273904      |RECORD   |X,GAP        |GRANTED    |10       |
|INNODB|140622813691256:118:4:6:140622732274248|25160669             |104      |94      |ed_test      |user       |NULL          |NULL             |PRIMARY   |140622732274248      |RECORD   |X            |WAITING    |11       |
+------+---------------------------------------+---------------------+---------+--------+-------------+-----------+--------------+-----------------+----------+---------------------+---------+-------------+-----------+---------+

```

发现两个事务锁定同一个记录11.

1. 事务1的insert产生了一个插入意向锁，事务2的insert也产生了一个插入意向锁（不会被互相锁住，因为数据行并不冲突）
2. 此时事务1再进行update语句，因未走索引，导致扫全表，而在扫到事务2插入那条数据时，行锁与插入意向锁冲突了，导致事务1需要等待事务2释放插入意向锁而进行等待。
3. 事务2在进行update时，也同样需要扫全表，但是全表都被事务1的update锁住了，事务2需要等待 `等待事务2释放插入意向锁的` 事务1 的行锁 释放，因此发生了死锁

如果我们使用索引字段age来更新，就不会出现死锁



# 2. InnoDB避免死锁：

死锁在行锁及事务场景下很难完全消除，但可以通过表设计和SQL调整等措施减少锁冲突和死锁，包括：

- 尽量使用较低的隔离级别，比如如果发生了间隙锁，你可以把会话或者事务的事务隔离级别更改为 RC(read committed)级别来避免，但此时需要把 binlog_format 设置成 row 或者 mixed 格式
- 精心设计索引，并尽量使用索引访问数据，使加锁更精确，从而减少锁冲突的机会；
- 选择合理的事务大小，小事务发生锁冲突的几率也更小；
- 给记录集显示加锁时，最好一次性请求足够级别的锁。比如要修改数据的话，最好直接申请排他锁，而不是先申请共享锁，修改时再请求排他锁，这样容易产生死锁；
- 不同的程序访问一组表时，应尽量约定以相同的顺序访问各表，对一个表而言，尽可能以固定的顺序存取表中的行。这样可以大大减少死锁的机会；

如果出现死锁，可以用 `SHOW ENGINE INNODB STATUS` 命令来确定最后一个死锁产生的原因。返回结果中包括死锁相关事务的详细信息，如引发死锁的 SQL 语句，事务已经获得的锁，正在等待什么锁，以及被回滚的事务等。据此可以分析死锁产生的原因和改进措施。

但是对于 RC/RR 模式下 ，insert 遇到唯一键冲突的时候的死锁不可避免。需要开发在设计表结构的时候 减少 unique 索引设计。

RR事务隔离级别和GAP锁是导致死锁的常见原因，但是业务逻辑设计不合理也会出发死锁,本文的案例通过修改业务逻辑最终将死锁解决。

# 3. 参考资料

https://mp.weixin.qq.com/s/9H4e2c2iqB1Iqz4npXGrjA

https://mp.weixin.qq.com/s/1Jd-8d-hZgGlAkvjWmO5kw

https://mp.weixin.qq.com/s/HH5m-oG9RZrEdxAk-cSreg