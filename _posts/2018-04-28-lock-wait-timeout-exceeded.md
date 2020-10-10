---
layout: post
title: MySQL锁（4）-Lock wait timeout exceeded
date: 2018-04-28
categories:
    - MySQL
comments: true
permalink: lock-wait-timeout-exceeded.html
---

`Lock wait timeout exceeded`与`Dead Lock`是不一样。

- `Lock wait timeout exceeded`：后提交的事务等待前面处理的事务释放锁，但是在等待的时候超过了mysql的锁等待时间，就会引发这个异常。
- `Dead Lock`：两个事务互相等待对方释放相同资源的锁，从而造成的死循环，就会引发这个异常。

还有一个要注意的是`innodb_lock_wait_timeout`与`lock_wait_timeout`也是不一样的。

- `innodb_lock_wait_timeout`：innodb的dml操作的行级锁的等待时间
- `lock_wait_timeout`：数据结构ddl操作的锁的等待时间

默认 lock 超时时间 50s

```

mysql> show variables like '%lock_wait%';
+--------------------------+----------+
| Variable_name            | Value    |
+--------------------------+----------+
| innodb_lock_wait_timeout | 50       |
| lock_wait_timeout        | 31536000 |
+--------------------------+----------+
2 rows in set (0.00 sec)

```

**临时解决方案**

通过`show full processlist`找到所有线程，在查询当前事务、锁占用情况，杀掉出问题的线程



MySQL日志文件系统的组成：

- error 日志：记录启动、运行或停止 mysqld 时出现的问题，默认开启。
- general 日志：通用查询日志，记录所有语句和指令，开启数据库会有 5% 左右性能损失。
- binlog 日志：二进制格式，记录所有更改数据的语句，主要用于 slave 复制和数据恢复。
- slow 日志：记录所有执行时间超过 long_query_time 秒的查询或不使用索引的查询，默认关闭。
- Innodb日志：innodb redo log、undo log，用于恢复数据和撤销操作。

为了定位`Lock wait timeout exceeded`的问题，我们需要检查general日志和slow日志

> general日志的开启要谨慎，会对数据库有一定的性能损耗

查看general日志的配置

```
mysql> show variables like '%general%';
+------------------+----------------------------------------------------+
| Variable_name    | Value                                              |
+------------------+----------------------------------------------------+
| general_log      | OFF                                                |
| general_log_file | /server/data/mysql/mysql-8.0.20/edgar-server-1.log |
+------------------+----------------------------------------------------+
2 rows in set (0.00 sec)

```

通过set命令设置全局开启，在MySQL重启后会失效

```

mysql> set global general_log = true;
Query OK, 0 rows affected (0.04 sec)

```

```
# cat /server/data/mysql/mysql-8.0.20/edgar-server-1.log
/usr/local/mysql/mysql-8.0.20/bin/mysqld, Version: 8.0.20 (MySQL Community Server - GPL). started with:
Tcp port: 3306  Unix socket: /usr/local/mysql/mysql-8.0.20/mysql.sock
Time                 Id Command    Argument
2020-09-08T05:37:49.335773Z        41 Query     begin
2020-09-08T05:38:01.691141Z        41 Query     INSERT INTO ed_test.user (id, username, nickname, gender, age) VALUES (7, 'username6', 'nickname6', 1, 45)
2020-09-08T05:38:08.470821Z        41 Query     select * from user where id = 5 for update
2020-09-08T05:38:15.520403Z        44 Query     begin
2020-09-08T05:38:19.002574Z        44 Query     select * from user where id = 10 for update
2020-09-08T05:38:22.334556Z        44 Query     INSERT INTO ed_test.user (id, username, nickname, gender, age) VALUES (7, 'username6', 'nickname6', 1, 45)
```
