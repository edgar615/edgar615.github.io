---
layout: post
title: MySQL锁-innodb锁（part1）
date: 2019-06-09
categories:
    - MySQL
comments: true
permalink: innodb-lock.html
---

InnoDB是基于事务，用来锁定的对象是数据库中的对象，如表、页、行。一般锁仅在事务commit或rollback后进行释放（不同事务隔离级别释放的时间可能不同）
可以通过`innodb_trx`,`innodb_locks`, `innodb_lock_waits`来观察锁的信息

**表锁：**操作对象是数据表。Mysql大多数锁策略都支持，是系统开销最低但并发性最低的一个锁策略。事务t对整个表加读锁，则其他事务可读不可写，若加写锁，则其他事务增删改都不行。

**行级锁：**操作对象是数据表中的一行。是MVCC技术用的比较多的。行级锁对系统开销较大，但处理高并发较好。

**InnoDB只有通过索引条件检索数据才使用行级锁，否则，InnoDB将使用表锁，也就是说，InnoDB的行锁是基于索引的**

- 记录锁
- 共享/排它锁
- 意向锁
- 间隙锁
- 临键锁
- 插入意向锁
- 自增锁

# 记录锁
记录锁(Record-Lock)是锁住记录的，这里要说明的是这里**锁住的是索引记录**，而不是我们真正的数据记录：
- 如果锁的是非主键索引，会在自己的索引上面加锁之后然后再去主键上面加锁锁住。
- 如果没有表上没有索引(包括没有主键)，则会使用隐藏的主键索引进行加锁。
- 如果要锁的列没有索引，则会进行全表记录加锁。


# 共享/排他锁
**共享锁：**也叫读锁、S锁（Shared Locks），若事务T对数据对象A加上S锁，则事务T可以读A但不能修改A，其他事务只能再对A加S锁，而不能加X锁，直到T释放A上的S 锁。这保证了其他事务可以读A，但在T释放A上的S锁之前不能对A做任何修改。

**排他锁：**又称写锁、X锁(Exclusive Locks)。若事务T对数据对象A加上X锁，事务T可以读A也可以修改A，其他事务不能再对A加任何锁，直到T释放A上的锁。这保证了其他事务在T释放A上的锁之前不能再读取和修改A。

所以我们说S锁和S锁是兼容的，S锁和X锁是不兼容的，X锁和X锁也是不兼容的

## 读操作
对读取的记录加S锁：
```
SELECT ... LOCK IN SHARE MODE;
```
对读取的记录加X锁：
```
SELECT ... FOR UPDATE;
```

## 写操作

**DELETE**

对一条记录做DELETE操作的过程其实是先在B+树中定位到这条记录的位置，然后获取一下这条记录的X锁，然后再执行delete mark操作。我们也可以把这个定位待删除记录在B+树中位置的过程看成是一个获取X锁的锁定读。

**UPDATE**

在对一条记录做UPDATE操作时分为三种情况：

1. 如果未修改该记录的键值并且被更新的列占用的存储空间在修改前后未发生变化，则先在B+树中定位到这条记录的位置，然后再获取一下记录的X锁，最后在原记录的位置进行修改操作。其实我们也可以把这个定位待修改记录在B+树中位置的过程看成是一个获取X锁的锁定读。
2. 如果未修改该记录的键值并且至少有一个被更新的列占用的存储空间在修改前后发生变化，则先在B+树中定位到这条记录的位置，然后获取一下记录的X锁，将该记录彻底删除掉（就是把记录彻底移入垃圾链表），最后再插入一条新记录。这个定位待修改记录在B+树中位置的过程看成是一个获取X锁的锁定读，新插入的记录由INSERT操作提供的隐式锁进行保护。
3. 如果修改了该记录的键值，则相当于在原记录上做DELETE操作之后再来一次INSERT操作，加锁操作就需要按照DELETE和INSERT的规则进行了。

**INSERT**

一般情况下，新插入一条记录的操作并不加锁，InnoDB通过隐式锁来保护这条新插入的记录在本事务提交前不被别的事务访问，当然在一些特殊情况下INSERT操作也是会获取锁的。


在对某个表执行`SELECT`、`INSERT`、`DELETE`、`UPDATE`语句时，InnoDB存储引擎是不会为这个表添加表级别的S锁或者X锁的。

另外，在对某个表执行一些诸如`ALTER TABLE`、`DROP TABLE`这类的DDL语句时，其他事务对这个表并发执行诸如`SELECT`、`INSERT`、`DELETE`、`UPDATE`的语句会发生阻塞，同理，某个事务中对某个表执行`SELECT`、`INSERT`、`DELETE`、`UPDATE`语句时，在其他会话中对这个表执行DDL语句也会发生阻塞。这个过程其实是通过在server层使用一种称之为元数据锁（Metadata Locks，简称MDL）来实现的，一般情况下也不会使用InnoDB存储引擎自己提供的表级别的S锁和X锁。

> 完全摘自“我们都是小青蛙”的文章

# 意向锁

InnoDB支持多粒度锁定，这种锁定允许事务在行级上的锁和表级上的锁同时存在。为了支持在不同粒度上的加锁操作，InnoDB提供了意向锁

我们对一个表可以加S锁和X锁。如果一个事务给表加了S锁，别的事务可以继续获得该表的S锁，也可以继续获得该表中的某些记录的S锁，但不能获取该表的X锁和某些记录的X锁；如果一个事务给表加了X锁，别的事务不可以继续获得该表的S锁和X锁，也不可以继续获得该表中的某些记录的S锁和X锁。

如果我们想获取表上的S锁，首先要保证表和表中的记录没有X锁，同样，如果我们想获取表上的X锁，我们要保证表和表中没有S锁或X锁。InnoDB通过意向锁来避免遍历表中的数据判断记录是否存在已经上锁的记录。

**IS锁：**意向共享锁(Intention Shared Lock)，当事务准备在某条记录上加S锁时，需要先在表级别加一个IS锁

**IX锁：**意向排他锁(Intention Exclusive Lock)，当事务准备在某条记录上加X锁时，需要先在表级别加一个IX锁

IS、IX锁是表级锁，而InnoDB支持的是行级别的锁，因此意向锁是不会阻塞除全表扫描以外的任何请求。所以IS、IX是兼容的。这一块可能会让人理解有点绕。

**IX，IS是表级锁，不会和行级的X，S锁发生冲突。只会和表级的X，S发生冲突**，简单来说，IX,IS是用于事务获取表锁时候判断数据行是否存在锁。它并不会影响行锁

示例

事务 A 先获取了某一行的X锁：
```
SELECT * FROM users WHERE id = 6 FOR UPDATE;
```
事务 B 想要获取表的S锁，检测到A持有IX锁，该请求被阻塞
```
LOCK TABLES users READ;
```
事务 C 也想获取 表中某一行的X锁，意向锁兼容，ID=5不存在锁，所以可以获取到X锁：
```
SELECT * FROM users WHERE id = 5 FOR UPDATE;
```

# 间隙锁GAP

当我们用范围条件检索数据而不是相等条件检索数据，并请求共享或排他锁时，InnoDB会给符合范围条件的已有数据记录的**索引项**加锁；对于键值在条件范围内但并不存在的记录，叫做“间隙（GAP)”。InnoDB也会对这个“间隙”加锁，这种锁机制就是所谓的间隙锁。

值得注意的是：**间隙锁只会在Repeatable read隔离级别下使用** 为什么呢?^_^

例子：假如emp表中只有101条记录，其empid的值分别是1,2,...,100,101
```
select * from emp where empid > 100 for update;
```
上面是一个范围查询，InnoDB不仅会对符合条件的empid值为101的记录加锁，也会对empid大于101（这些记录并不存在）的“间隙”加锁。此时在另一个事务插入ID为102的数据会卡住，等待X锁的释放。

InnoDB使用间隙锁的目的有两个：

- 为了防止幻读，Repeatable read隔离级别下再通过GAP锁即可避免了幻读，因为在事务在第一次执行读取操作时，那些幻影记录尚不存在，我们无法给这些幻影记录加上锁
- 满足恢复和复制的需要MySQL的恢复机制要求：在一个事务未提交前，其他并发事务不能插入满足其锁定条件的任何记录，也就是不允许出现幻读

> 间隙锁要么锁住索引记录中间的值，要么锁住第一个索引记录前面的值或者最后一个索引记录后面的值。

## 案例1

准备数据

```
create table emp(
empno smallint(5) unsigned not null auto_increment,
ename varchar(30) not null,
deptno smallint(5) unsigned not null,
job varchar(30) not null,
primary key(empno),
key idx_emp_info(deptno,ename)
)engine=InnoDB charset=utf8;

insert into emp values(1,'zhangsan',1,'CEO'),(2,'lisi',2,'CFO'),(3,'wangwu',3,'CTO'),(4,'jeanron100',3,'Enginer');
```

事务A
```
start transaction;
begin;
SELECT * FROM emp WHERE empno BETWEEN 10 AND 30 FOR UPDATE;
```
事务B
```
start transaction;
begin;
-- 会被卡住，等待事务A的提交
insert emp(ename, deptno,job) values('zhangsan', 1, 'CEO');
```

## 案例2

表中的ID最大为17，缺少10，11，13，14

```
mysql> select * from emp;
+-------+------------+--------+---------+
| empno | ename      | deptno | job     |
+-------+------------+--------+---------+
|     1 | zhangsan   |      1 | CEO     |
|     2 | lisi       |      2 | CFO     |
|     3 | wangwu     |      3 | CTO     |
|     4 | jeanron100 |      3 | Enginer |
|     5 | zhangsan   |      1 | CEO     |
|     6 | zhangsan   |      1 | CEO     |
|     7 | zhangsan   |      1 | CEO     |
|     8 | zhangsan   |      1 | CEO     |
|     9 | zhangsan   |      1 | CEO     |
|    12 | zhangsan   |      1 | CEO     |
|    15 | zhangsan   |      1 | CEO     |
|    16 | zhangsan   |      1 | CEO     |
|    17 | zhangsan   |      1 | CEO     |
+-------+------------+--------+---------+
13 rows in set (0.06 sec)
```
事务A 
```
mysql> start transaction;
mysql> begin;
mysql> SELECT * FROM emp WHERE empno = 19 FOR UPDATE;
Empty set
```
事务B会卡住
```
insert emp(ename, deptno,job) values('zhangsan', 1, 'CEO');
```
事务A的检索条件empno=19,向左取得最靠近的值17作为左区间，向右由于没有记录因此取得无穷大作为右区间，因此，**事务A的间隙锁的范围（17，无穷大）**

## 案例3

事务A
```
SELECT * FROM emp WHERE empno = 12 FOR UPDATE;
```
事务B
```
-- 执行成功
insert emp(ename, deptno,job) values('zhangsan', 1, 'CEO');
update emp set ename = 'test' where empno = 9;
update emp set ename = 'test' where empno = 10;
update emp set ename = 'test' where empno = 13;
update emp set ename = 'test' where empno = 15;
insert emp(empno,ename, deptno,job) values(13,'zhangsan', 1, 'CEO');
insert emp(empno,ename, deptno,job) values(10,'zhangsan', 1, 'CEO');
```
**记录锁 12**

## 案例4

事务A
```
SELECT * FROM emp WHERE empno = 13 FOR UPDATE;
```
事务B
```
-- 执行成功
insert emp(empno,ename, deptno,job) values(10,'zhangsan', 1, 'CEO');
insert emp(empno,ename, deptno,job) values(18,'zhangsan', 1, 'CEO');
-- 卡住
insert emp(empno,ename, deptno,job) values(14,'zhangsan', 1, 'CEO');
```
**间隙锁(12,15)**

## 案例5

事务A
```
SELECT * FROM emp WHERE empno = 13 FOR UPDATE;
```
事务B
```
-- 执行成功
update emp set empno = 21 where empno = 1;
-- 卡住
update emp set empno = 13 where empno = 21;
```

总结一下：**间隙锁的提出仅仅是为了防止插入幻影记录而提出的（防止其他事务在间隔中插入数据），RR隔离级别就是通过间隙锁来减少幻读的发生**


# 临键锁
Next-Key 可以理解为一种特殊的间隙锁，也可以理解为一种特殊的算法。通过临建锁可以解决幻读的问题。 每个数据行上的**非唯一索引列**上都会存在一把临键锁，当某个事务持有该数据行的临键锁时，会锁住一段**左开右闭区间**的数据。需要强调的一点是，InnoDB 中行级锁是基于索引实现的，临键锁只与非唯一索引列有关，在唯一索引列（包括主键列）上不存在临键锁。

> Next-Key Lock可以说是记录锁(Record Lock)和间隙锁（Gap Lock）的组合，既封锁了”缝隙”，又封锁了索引本身。

假设有如下表：
```
create table next_key_test(
id smallint(5) unsigned not null auto_increment,
name varchar(30) not null,
age smallint(5) unsigned not null,
primary key(id),
key idx_age(age)
)engine=InnoDB charset=utf8;

insert next_key_test(id, name, age) values(1, 'lee', 10),(3, 'lee', 24),(5, 'lee', 32),(7, 'lee', 45);
```
该表中 age 列潜在的临键锁有：
```
(-∞, 10],
(10, 24],
(24, 32],
(32, 45],
(45, +∞],
```

事务A
```
SELECT * FROM next_key_test WHERE age = 24 FOR UPDATE;
```
事务B
```
-- 卡住
INSERT INTO next_key_test VALUES(100, 'Ezreal', 26);
INSERT INTO next_key_test VALUES(101, 'Ezreal', 10);
-- 成功
INSERT INTO next_key_test VALUES(102, 'Ezreal', 32);
INSERT INTO next_key_test VALUES(103, 'Ezreal', 33);
INSERT INTO next_key_test VALUES(104, 'Ezreal', 9);
```
事务 A 在对 age 为 24 的列进行 UPDATE 操作的同时，也获取了 (24, 32] 这个区间内的临键锁。

**注意：临键锁只与非唯一索引列有关，在唯一索引列（包括主键列）上不存在临键锁。**

在根据非唯一索引对记录行进行` UPDATE` , `FOR UPDATE`, `LOCK IN SHARE MODE` 操作时，InnoDB 会获取该记录行的 临键锁 ，并同时获取该记录行下一个区间的间隙锁。**即：临键锁也包括间隙锁**

但是根据上面的说法，不是应该锁住(10,32]区间，实际测试却锁住的是[10,32)，why？

# 插入意向锁
一个事务在插入一条记录时需要判断一下插入位置是不是被别的事务加了间隙锁，如果有的话，插入操作需要等待，直到拥有间隙锁的那个事务提交。
**插入意向锁(Insert Intention Locks)**它是插入之前由插入操作设置的间隙锁的一种类型。这个锁表示插入的意图，如果插入到同一索引间隙中的多个事务没有插入到间隙中的同一位置，那么它们就不需要等待对方。如果是同一位置，但不要因为行不冲突而相互阻塞。

**插入意向锁实际上是一个间隙锁，不是意向锁，在insert时产生**

- 普通的间隙锁不允许 在 （上一条记录，本记录） 范围内插入数据
- 插入意向锁允许 在 （上一条记录，本记录） 范围内插入数据

插入意向锁的作用是为了提高并发插入的性能， 多个事务 同时写入 不同数据至同一索引范围（区间）内，并不需要等待其他事务完成，不会发生锁等待。

**插入的过程**

假设现在有记录 10， 30， 50， 70 ；且为主键 ，需要插入记录 25 。

1. 找到 小于等于25的记录 ，这里是 10
2. 找到 记录10的下一条记录 ，这里是 30
3. 判断 下一条记录30 上是否有锁
- 判断 30 上面如果 没有锁 ，则可以插入
- 判断 30 上面如果有Record Lock，则可以插入
-  判断 30 上面如果有Gap Lock/Next-Key Lock，则无法插入，因为锁的范围是 (10, 30) /（10, 30] ；在30上增加insert intention lock（ 此时处于waiting状态），当 Gap Lock / Next-Key Lock 释放时，等待的事务（ transaction）将被唤醒 ，此时 记录30 上才能获得 insert intention lock ，然后再插入记录25

注意：一个事务 insert 25 且没有提交，另一个事务 delete 25 时，记录25上会有 Record Lock

总结一下: **如果即将插入的间隙已经被其他事务加了间隙锁，那么本次INSERT操作会阻塞，并且当前事务会在该间隙上加一个插入意向锁**

# 自增锁

在InnoDB存储引擎的内存结构中，对每个含有自增长值的表都有一个自增长计数器。当对含有自增长的计数器的表进行插入操作时，这个计数器会被初始化，执行如下的语句来得到计数器的值：

```
select max(auto_inc_col) from t for update;
```

插入操作会依据这个自增长的计数器值加1赋予自增长列。这个实现方式称为AUTO-INC Locking。**这种锁其实是采用一种特殊的表锁机制，为了提高插入的性能，锁不是在一个事务完成后才释放，而是在完成对自增长值插入的SQL语句后立即释放**。

虽然AUTO-INC Locking从一定程度上提高了并发插入的效率，但还是存在一些性能上的问题。首先，对于有自增长值的列的并发插入性能较差，事务必须等待前一个插入的完成，虽然不用等待事务的完成。其次，对于INSERT….SELECT的大数据的插入会影响插入的性能，因为另一个事务中的插入会被阻塞。

InnoDB存储引擎中提供了一种轻量级互斥量的自增长实现机制，这种机制大大提高了自增长值插入的性能。并且从该版本开始，InnoDB存储引擎提供了一个参数`innodb_autoinc_lock_mode`来控制自增长的模式，该参数的默认值为1。在继续讨论新的自增长实现方式之前，需要对自增长的插入进行分类。如下说明：

**insert-like**：指所有的插入语句，如INSERT、REPLACE、INSERT…SELECT，REPLACE…SELECT、LOAD DATA等。

**simple inserts**：指能在插入前就确定插入行数的语句，这些语句包括INSERT、REPLACE等。需要注意的是：simple inserts不包含INSERT…ON DUPLICATE KEY UPDATE这类SQL语句。

**bulk inserts**：指在插入前不能确定得到插入行数的语句，如INSERT…SELECT，REPLACE…SELECT，LOAD DATA。

**mixed-mode inserts**：指插入中有一部分的值是自增长的，有一部分是确定的。入INSERT INTO t1(c1,c2) VALUES(1,’a’),(2,’a’),(3,’a’)；也可以是指INSERT…ON DUPLICATE KEY UPDATE这类SQL语句。

接下来分析参数innodb_autoinc_lock_mode以及各个设置下对自增长的影响，其总共有三个有效值可供设定，即0、1、2，具体说明如下：

**0**：这是MySQL 5.1.22版本之前自增长的实现方式，即通过表锁的AUTO-INC Locking方式，因为有了新的自增长实现方式，0这个选项不应该是新版用户的首选了。

**1**：这是该参数的默认值，对于”simple inserts”，该值会用互斥量（mutex）去对内存中的计数器进行累加的操作。对于”bulk inserts”，还是使用传统表锁的AUTO-INC Locking方式。在这种配置下，如果不考虑回滚操作，对于自增值列的增长还是连续的。并且在这种方式下，statement-based方式的replication还是能很好地工作。需要注意的是，如果已经使用AUTO-INC Locking方式去产生自增长的值，而这时需要再进行”simple inserts”的操作时，还是需要等待AUTO-INC Locking的释放。

**2**：在这个模式下，对于所有”INSERT-LIKE”自增长值的产生都是通过互斥量，而不是AUTO-INC Locking的方式。显然，这是性能最高的方式。然而，这会带来一定的问题，因为并发插入的存在，在每次插入时，自增长的值可能不是连续的。此外，最重要的是，基于Statement-Base Replication会出现问题。因此，使用这个模式，任何时候都应该使用row-base replication。这样才能保证最大的并发性能及replication主从数据的一致。

> 完全摘自《MySQL技术内幕  InnoDB存储引擎》

假如我们插入的数据中有AUTO_INCREMENT列，InnoDB在RR（Repeatable Read）隔离级别下，能解决幻读问题。
```
    //事务A先执行，还未提交：
    insert into t(name) values(xxx);
    //事务B后执行：
    insert into t(name) values(ooo);
```

1. 事务A先执行insert，会得到一条(4, xxx)的记录，由于是自增列，InnoDB会自动增长，注意此时事务并未提交；
2. 事务B后执行insert，假设不会被阻塞，那会得到一条(5, ooo)的记录；
3. 事务A继续insert：会得到一条(6, xxoo)的记录。
4. 事务A再select：select * from t where id>3，在mvvc中当然可以得到(4, xxx)和(6, xxoo)

这对于事务A来说，就很奇怪了，对于AUTO_INCREMENT的列，连续插入了两条记录，ID缺不是连续的

自增锁是一种特殊的表级别锁（table-level lock），专门针对事务插入AUTO_INCREMENT类型的列。最简单的情况，如果一个事务正在往表中插入记录，所有其他事务的插入必须等待，以便第一个事务插入的行，是连续的主键值。与此同时，InnoDB提供了innodb_autoinc_lock_mode配置，可以调节与改变该锁的模式与行为。

# insert|select|update|delete的锁

## select

普通的select是快照读，而select ... for update或select ... in share mode则会根据情况加不同的锁
- 如果在唯一索引上用唯一的查询条件时（ where id=1），加记录锁
- 否则，其他的查询条件和索引条件，加间隙锁（BETWEEN AND ）或Next-Key 锁（可重复隔离级别）

## update与delete

- 如果在唯一索引上使用唯一的查询条件来update/delete，加记录锁
- 否则，符合查询条件的索引记录之前，都会加Next-Key 锁

注：**如果update的是聚集索引，则对应的普通索引记录也会被隐式加锁，这是由InnoDB索引的实现机制决定的：普通索引存储PK的值，检索普通索引本质上要二次扫描聚集索引。**

## insert

insert和update与delete不同，它会用排它锁封锁被插入的索引记录，同时，会在插入区间加插入意向锁，但这个并不会真正封锁区间，也不会阻止相同区间的不同KEY插入。


# 分析行锁定

通过检查InnoDB_row_lock 状态变量分析系统上的行锁的争夺情况` show status like 'innodb_row_lock%'`

```
mysql> show status like 'innodb_row_lock%';
+-------------------------------+----------+
| Variable_name                 | Value    |
+-------------------------------+----------+
| Innodb_row_lock_current_waits | 0        |
| Innodb_row_lock_time          | 95856315 |
| Innodb_row_lock_time_avg      | 309      |
| Innodb_row_lock_time_max      | 51959    |
| Innodb_row_lock_waits         | 310172   |
+-------------------------------+----------+
5 rows in set (0.03 sec)
```

- innodb_row_lock_current_waits: 当前正在等待锁定的数量
- innodb_row_lock_time: 从系统启动到现在锁定总时间长度；非常重要的参数
- innodb_row_lock_time_avg: 每次等待所花平均时间；非常重要的参数
- innodb_row_lock_time_max: 从系统启动到现在等待最常的一次所花的时间
- innodb_row_lock_waits: 系统启动后到现在总共等待的次数；非常重要的参数。直接决定优化的方向和策略

# 行锁优化

- 尽可能让所有数据检索都通过索引来完成，避免无索引行或索引失效导致行锁升级为表锁
- 尽可能避免间隙锁带来的性能下降，减少或使用合理的检索范围
- 尽可能减少事务的粒度，比如控制事务大小，而从减少锁定资源量和时间长度，从而减少锁的竞争等，提供性能
- 尽可能低级别事务隔离，隔离级别越高，并发的处理能力越低。

# 死锁
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

## InnoDB避免死锁：

- 为了在单个InnoDB表上执行多个并发写入操作时避免死锁，可以在事务开始时通过为预期要修改的每个元祖（行）使用`SELECT ... FOR UPDATE`语句来获取必要的锁，即使这些行的更改语句是在之后才执行的。
- 在事务中，如果要更新记录，应该直接申请足够级别的锁，即排他锁，而不应先申请共享锁、更新时再申请排他锁，因为这时候当用户再申请排他锁时，其他事务可能又已经获得了相同记录的共享锁，从而造成锁冲突，甚至死锁如果事务需要修改或锁定多个表，则应在每个事务中以相同的顺序使用加锁语句。 
- 在应用中，如果不同的程序会并发存取多个表，应尽量约定以相同的顺序来访问表，这样可以大大降低产生死锁的机会
- 通过`SELECT ... LOCK IN SHARE MODE`获取行的读锁后，如果当前事务再需要对该记录进行更新操作，则很有可能造成死锁。
- 改变事务隔离级别

如果出现死锁，可以用 SHOW INNODB STATUS 命令来确定最后一个死锁产生的原因。返回结果中包括死锁相关事务的详细信息，如引发死锁的 SQL 语句，事务已经获得的锁，正在等待什么锁，以及被回滚的事务等。据此可以分析死锁产生的原因和改进措施。

# 什么时候使用表锁

InnoDB默认采用行锁，在未使用索引字段查询时升级为表锁。MySQL这样设计并不是给你挖坑。它有自己的设计目的。即便你在条件中使用了索引字段，MySQL会根据自身的执行计划，考虑是否使用索引(所以explain命令中会有possible_key 和 key)。如果MySQL认为全表扫描效率更高，它就不会使用索引，这种情况下InnoDB将使用表锁，而不是行锁。因此，在分析锁冲突时，别忘了检查SQL的执行计划，以确认是否真正使用了索引。

- 第一种情况：全表更新。事务需要更新大部分或全部数据，且表又比较大。若使用行锁，会导致事务执行效率低，从而可能造成其他事务长时间锁等待和更多的锁冲突。
- 第二种情况：多表查询。事务涉及多个表，比较复杂的关联查询，很可能引起死锁，造成大量事务回滚。这种情况若能一次性锁定事务涉及的表，从而可以避免死锁、减少数据库因事务回滚带来的开销。

# 参考资料

https://mp.weixin.qq.com/s/FSyE7Tz5A-Rc1bkC-tDqvA

https://mp.weixin.qq.com/s/wGOxro3uShp2q5w97azx5A

https://zhuanlan.zhihu.com/p/29150809

《MySQL 是怎样运行的：从根儿上理解 MySQL》