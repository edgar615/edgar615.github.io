---
layout: post
title: MySQL事务（4）- 半一致性读
date: 2018-09-04
categories:
    - MySQL
comments: true
permalink: mysql-transaction-semi-consistent.html
---

> 复制自https://zhuanlan.zhihu.com/p/150248692

# 1. **什么是半一致性读？**

先看下官方的描述：

- 是一种用在 Update 语句中的读操作（一致性读）的优化，是在 RC 事务隔离级别下与一致性读的结合。
- 当 Update 语句的 where 条件中匹配到的记录已经上锁，会再次去 InnoDB 引擎层读取对应的行记录，判断是否真的需要上锁（第一次需要由 InnoDB 先返回一个最新的已提交版本）。
- 只在 RC 事务隔离级别下或者是设置了 innodb_locks_unsafe_for_binlog=1 的情况下才会发生。
- innodb_locks_unsafe_for_binlog 参数在 8.0 版本中已被去除（可见，这是一个可能会导致数据不一致的参数，官方也不建议使用了）。

> 所谓半一致性读就是，一个update语句，如果读到一行已经加锁的记录，此时InnoDB返回记录最近提交的版本，由MySQL上层判断此版本是否满足update的where条件。若满足(需要更新)，则MySQL会重新发起一次读操作，此时会读取行的最新版本(并加锁)！

# 2. 案例1

我们先通过 2 个测试案例来观察半一致性读会对事务产生哪些影响。

**准备数据**

```
mysql> desc t;
+-------+------+------+-----+---------+-------+
| Field | Type | Null | Key | Default | Extra |
+-------+------+------+-----+---------+-------+
| id    | int  | YES  |     | NULL    |       |
| sal   | int  | YES  |     | NULL    |       |
+-------+------+------+-----+---------+-------+
2 rows in set (0.00 sec)

mysql> select * from t;
+------+------+
| id   | sal  |
+------+------+
|    1 |  100 |
|    2 |  200 |
|    3 |  300 |
|    4 |  400 |
|    5 |  500 |
|    6 |  600 |
|    7 |  700 |
|    8 |  800 |
|    9 |  900 |
|   10 | 1000 |
+------+------+
10 rows in set (0.00 sec)

mysql> show variables like 'transaction_isolation';
+-----------------------+----------------+
| Variable_name         | Value          |
+-----------------------+----------------+
| transaction_isolation | READ-COMMITTED |
+-----------------------+----------------+
1 row in set (0.00 sec)

# 设置参数innodb_status_output_locks=on，否则看不到IX锁
mysql> show variables like 'innodb_status_output_locks';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| innodb_status_output_locks | ON    |
+----------------------------+-------+
1 row in set (0.00 sec)
```

**session1**

```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t where id > 3 and id < 6 for update;
+------+------+
| id   | sal  |
+------+------+
|    4 |  400 |
|    5 |  500 |
+------+------+
2 rows in set (0.00 sec)
```

查看事务状态

```
mysql> show engine innodb status;
...
------------
TRANSACTIONS
------------
Trx id counter 6699539
Purge done for trx's n:o < 6699526 undo n:o < 0 state: running but idle
History list length 3
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421163334236736, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421163334235880, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 6699538, ACTIVE 3 sec
2 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 9, OS thread handle 139688312157952, query id 36 localhost root starting
show engine innodb status
TABLE LOCK table `ds0`.`t` trx id 6699538 lock mode IX
RECORD LOCKS space id 8692 page no 4 n bits 80 index GEN_CLUST_INDEX of table `ds0`.`t` trx id 6699538 lock_mode X locks rec but not gap
Record lock, heap no 5 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 6; hex 00000048b903; asc    H  ;;
 1: len 6; hex 000000663621; asc    f6!;;
 2: len 7; hex 82000001170110; asc        ;;
 3: len 4; hex 80000004; asc     ;;
 4: len 4; hex 80000190; asc     ;;

Record lock, heap no 6 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 6; hex 00000048b904; asc    H  ;;
 1: len 6; hex 000000663622; asc    f6";;
 2: len 7; hex 810000010e0110; asc        ;;
 3: len 4; hex 80000005; asc     ;;
 4: len 4; hex 800001f4; asc     ;;

...
```

线程9的事务6699538，获取到了1个表级插入意向锁IX，2个记录锁，对应id=4,id=5的这两条记录

**session2**

```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t where id = 7 for update;
# 等待锁

mysql> show engine innodb status;
...

------------
TRANSACTIONS
------------
Trx id counter 6699540
Purge done for trx's n:o < 6699526 undo n:o < 0 state: running but idle
History list length 0
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421163334236736, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421163334235880, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 6699539, ACTIVE 10 sec fetching rows
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 11, OS thread handle 139688309319424, query id 78 localhost root executing
select * from t where id = 7 for update
------- TRX HAS BEEN WAITING 10 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 8692 page no 4 n bits 80 index GEN_CLUST_INDEX of table `ds0`.`t` trx id 6699539 lock_mode X locks rec but not gap waiting
Record lock, heap no 5 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 6; hex 00000048b903; asc    H  ;;
 1: len 6; hex 000000663621; asc    f6!;;
 2: len 7; hex 82000001170110; asc        ;;
 3: len 4; hex 80000004; asc     ;;
 4: len 4; hex 80000190; asc     ;;

------------------
TABLE LOCK table `ds0`.`t` trx id 6699539 lock mode IX
RECORD LOCKS space id 8692 page no 4 n bits 80 index GEN_CLUST_INDEX of table `ds0`.`t` trx id 6699539 lock_mode X locks rec but not gap
RECORD LOCKS space id 8692 page no 4 n bits 80 index GEN_CLUST_INDEX of table `ds0`.`t` trx id 6699539 lock_mode X locks rec but not gap waiting
Record lock, heap no 5 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 6; hex 00000048b903; asc    H  ;;
 1: len 6; hex 000000663621; asc    f6!;;
 2: len 7; hex 82000001170110; asc        ;;
 3: len 4; hex 80000004; asc     ;;
 4: len 4; hex 80000190; asc     ;;

---TRANSACTION 6699538, ACTIVE 523 sec
2 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 9, OS thread handle 139688312157952, query id 79 localhost root starting
show engine innodb status
TABLE LOCK table `ds0`.`t` trx id 6699538 lock mode IX
RECORD LOCKS space id 8692 page no 4 n bits 80 index GEN_CLUST_INDEX of table `ds0`.`t` trx id 6699538 lock_mode X locks rec but not gap
Record lock, heap no 5 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 6; hex 00000048b903; asc    H  ;;
 1: len 6; hex 000000663621; asc    f6!;;
 2: len 7; hex 82000001170110; asc        ;;
 3: len 4; hex 80000004; asc     ;;
 4: len 4; hex 80000190; asc     ;;

Record lock, heap no 6 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 6; hex 00000048b904; asc    H  ;;
 1: len 6; hex 000000663622; asc    f6";;
 2: len 7; hex 810000010e0110; asc        ;;
 3: len 4; hex 80000005; asc     ;;
 4: len 4; hex 800001f4; asc     ;;
...
```

可以看到线程11的6699539事务正在请求并等待1个记录锁，id=4的这条记录。**为什么？**

innodb锁等待超时后再观察一次，线程11的事务6699539的事务仍然没有结束，对t表持有IX锁，并且仍然在等待id=4的行锁释放

**session3**

```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update t set sal = sal + 1 where id = 7;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```

在 Session 1 事务仍然未结束的情况下，Session 3 的事务未被阻塞，可以正常执行。

查看3个语句的执行计划

```
mysql> explain select * from t where id>3 and id<6 for update;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |    11.11 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> explain select * from t where id = 7 for update;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |    10.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> explain update t set sal=sal+1 where id=7;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | UPDATE      | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |   100.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

由于 t 表没有索引，执行计划必然是走全表扫描，也就是每条被读取到的记录，都会上行锁。那为何 Session 1 只锁了id=4，id=5 的这两条，并没有锁全表呢？而同样是请求 id=7 的记录，为何 Session 2 无法获取锁资源，Session 3  却能成功执行？也许大家从上面的锁分析可以很快得到结论，由于 Session 1 只占用了 id=4、id=5 的行锁，那么 Session 3  去请求 id=7 的自然不会有冲突（似乎挺有道理） 

那么 Session 2 对 id=7 的请求，为何会被锁定呢？

带着这些疑问，我们继续看第 2 个案例。

# 3. 案例2

这次 Session 1 执行的 Select 语句不带 where 条件

```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t for update;
+------+------+
| id   | sal  |
+------+------+
|    1 |  100 |
|    2 |  200 |
|    3 |  300 |
|    4 |  400 |
|    5 |  500 |
|    6 |  600 |
|    7 |  700 |
|    8 |  800 |
|    9 |  900 |
|   10 | 1000 |
+------+------+
10 rows in set (0.00 sec)

mysql> show engine innodb status;
------------
TRANSACTIONS
------------
Trx id counter 6699544
Purge done for trx's n:o < 6699543 undo n:o < 0 state: running but idle
History list length 0
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421163334239304, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421163334238448, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421163334236736, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421163334235880, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 6699543, ACTIVE 14 sec
2 lock struct(s), heap size 1136, 10 row lock(s)
MySQL thread id 9, OS thread handle 139688312157952, query id 350 localhost root starting
show engine innodb status
TABLE LOCK table `ds0`.`t` trx id 6699543 lock mode IX
RECORD LOCKS space id 8692 page no 4 n bits 80 index GEN_CLUST_INDEX of table `ds0`.`t` trx id 6699543 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 6; hex 00000048b900; asc    H  ;;
 1: len 6; hex 00000066361a; asc    f6 ;;
 2: len 7; hex 810000011a0110; asc        ;;
 3: len 4; hex 80000001; asc     ;;
 4: len 4; hex 80000064; asc    d;;

Record lock, heap no 3 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 6; hex 00000048b901; asc    H  ;;
 1: len 6; hex 00000066361b; asc    f6 ;;
 2: len 7; hex 820000010b0110; asc        ;;
 3: len 4; hex 80000002; asc     ;;
 4: len 4; hex 800000c8; asc     ;;

Record lock, heap no 4 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 6; hex 00000048b902; asc    H  ;;
 1: len 6; hex 000000663620; asc    f6 ;;
 2: len 7; hex 810000010c0110; asc        ;;
 3: len 4; hex 80000003; asc     ;;
 4: len 4; hex 8000012c; asc    ,;;

Record lock, heap no 5 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 6; hex 00000048b903; asc    H  ;;
 1: len 6; hex 000000663621; asc    f6!;;
 2: len 7; hex 82000001170110; asc        ;;
 3: len 4; hex 80000004; asc     ;;
 4: len 4; hex 80000190; asc     ;;

Record lock, heap no 6 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 6; hex 00000048b904; asc    H  ;;
 1: len 6; hex 000000663622; asc    f6";;
 2: len 7; hex 810000010e0110; asc        ;;
 3: len 4; hex 80000005; asc     ;;
 4: len 4; hex 800001f4; asc     ;;

Record lock, heap no 7 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 6; hex 00000048b905; asc    H  ;;
 1: len 6; hex 000000663623; asc    f6#;;
 2: len 7; hex 820000010d0110; asc        ;;
 3: len 4; hex 80000006; asc     ;;
 4: len 4; hex 80000258; asc    X;;

Record lock, heap no 8 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 6; hex 00000048b906; asc    H  ;;
 1: len 6; hex 000000663624; asc    f6$;;
 2: len 7; hex 810000010f0110; asc        ;;
 3: len 4; hex 80000007; asc     ;;
 4: len 4; hex 800002bc; asc     ;;

Record lock, heap no 9 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 6; hex 00000048b907; asc    H  ;;
 1: len 6; hex 000000663625; asc    f6%;;
 2: len 7; hex 820000010e0110; asc        ;;
 3: len 4; hex 80000008; asc     ;;
 4: len 4; hex 80000320; asc     ;;

Record lock, heap no 10 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 6; hex 00000048b908; asc    H  ;;
 1: len 6; hex 000000663626; asc    f6&;;
 2: len 7; hex 81000001100110; asc        ;;
 3: len 4; hex 80000009; asc     ;;
 4: len 4; hex 80000384; asc     ;;

Record lock, heap no 11 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 6; hex 00000048b909; asc    H  ;;
 1: len 6; hex 000000663627; asc    f6';;
 2: len 7; hex 820000010f0110; asc        ;;
 3: len 4; hex 8000000a; asc     ;;
 4: len 4; hex 800003e8; asc     ;;


```

线程9的6699543事务获得了1个IX表锁和10个X记录锁，即：把表中的10条记录都锁定了

> t表上没有索引，MySQL默认会创建GEN_CLUST_INDEX的聚簇索引，而语句没有加where条件，只能走全表

**session2**

```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t where id = 7 for update;
# 等待锁
```

与之前案例 1 相同，也是锁等待超时退出。

这次线程11的事务6699544从第1条记录就开始加锁了

```
---TRANSACTION 6699544, ACTIVE 41 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 11, OS thread handle 139688309319424, query id 368 localhost root executing
select * from t where id = 7 for update
------- TRX HAS BEEN WAITING 41 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 8692 page no 4 n bits 80 index GEN_CLUST_INDEX of table `ds0`.`t` trx id 6699544 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 6; hex 00000048b900; asc    H  ;;
 1: len 6; hex 00000066361a; asc    f6 ;;
 2: len 7; hex 810000011a0110; asc        ;;
 3: len 4; hex 80000001; asc     ;;
 4: len 4; hex 80000064; asc    d;;

------------------
TABLE LOCK table `ds0`.`t` trx id 6699544 lock mode IX
RECORD LOCKS space id 8692 page no 4 n bits 80 index GEN_CLUST_INDEX of table `ds0`.`t` trx id 6699544 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 6; hex 00000048b900; asc    H  ;;
 1: len 6; hex 00000066361a; asc    f6 ;;
 2: len 7; hex 810000011a0110; asc        ;;
 3: len 4; hex 80000001; asc     ;;
 4: len 4; hex 80000064; asc    d;;

```

**session3**

```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update t set sal = sal + 1 where id = 7;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

```

与案例 1 不同的是，这次 Update 语句也遭遇锁等待超时退出了。

# 4. **案例分析**

由于 t 表上不存在索引，3 个会话执行的语句都是**全表扫描**，在 RC  事务隔离级别下，这些语句都是需要发起当前读的操作（读取t表上最新的已提交事务版本），需要对读取到的全部记录加上记录锁（即行锁、也可称为  InnoDB 锁，大多数情况下，RC 隔离级别没有 Gap 锁，因此基本不太会出现 Next-Key 锁，对高并发场景比较友好）。

**案例 1**

- Session 1：开始需要对每条记录加锁，由于不需要维护可重复读，也不需要锁 Gap，当返回 MySQL Server 层通过 where 条件过滤后，最终只对 id=4、id=5 的记录加了锁。
- Session 2：从 id=1 开始读取记录并加锁，当读取到 id=4 的记录时，**由于 Session 1 先对 id=4  的记录上了锁，就无法再对其进行加锁操作，我们看到它一直在等待 id=4 的 X 锁，直到锁等待超时报错，为何是 id=4，而不是  id=5？因为是按聚簇索引一条条读取记录的，所以锁也需要一条条加，当上一条记录的锁资源没获取到，就不会对下一条记录加锁。**
- Session 3：同样地，最开始也需要对读取到的记录一条条加锁，由于 id=7 的记录与 id=4、id=5 上的行锁并不冲突，此处可以利用半一致性读对  Update 的优化特性，提前将 id=7 上的行锁释放掉了，因此 Update 不会被阻塞，事务得以正常执行。

**案例 2**

- Session 1：Select 语句没有用 where 条件，通过全表扫描访问到的所有记录都无法通过 MySQL Server 层过滤，因此将 t 表的全部记录都上了 X 锁。
- Session 2：由于 Session 1 已经将全部记录都上了 X 锁，Session 2 当前读的 Select 操作由于无法获取任何记录的 X 锁，就被阻塞了。
- Session 3：同样地，Session 1 持有的全记录 X 锁，使 Session 3 的 where 条件落到了匹配的区间内，表示 Session 1 对 id=7 的行确实需要更新，必须上锁，因此 Session 3 的 Update 被阻塞。

# 5. **总结**

在 RC 事务隔离级别下，Update 语句可以利用到半一致性读的特性，会多进行一次判断，当 where  条件匹配到的记录与当前持有锁的事务中的记录不冲突时，就会提前释放 InnoDB  锁，虽然这样做违背了二阶段加锁协议，但却可以减少锁冲突，提高事务并发能力，是一种很好的优化行为。