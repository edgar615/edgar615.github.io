---
layout: post
title: MySQL日志（5）- MySQL binlog事件类型
date: 2018-06-30
categories:
    - MySQL
comments: true
permalink: mysql-binlog-event.html
---

有项目要读取binlog，整理了下binlog的事件类型，做个记录


MySQL的binlog文件中记录的是对数据库的各种修改操作，以事件的形式来记录的。用来表示修改操作的数据结构是Log 
event。不同的修改操作对应的不同的log event。比较常用的几种log event有：Query event、Row event、Xid
 event等。其中Query event对应的是一条SQL语句，在DDL操作和STMT格式的binlog中用的比较多。Row 
event是个基础类，它的派生类有Row insert event、Row update event、Row delete 
event三种，分别对应ROW格式binlog的增、改、删操作。Xid 
event对应的是支持事务的commit操作，对于不支持事务的commit操作，记录的形式是Query 
event。其他还有一些event，比如Format log event、Rotate 
event等等，可以查看MySQL的官方文档了解更多相关信息。log event的种类一直在增加，比如MariaDB中新增的checkpoint
 event等。要MySQL本身就留有接口以便新增一个Log event，但是新增一个Log 
event时需要实现几个必要的方法函数，比如print、write、get_code_type等。binlog文件的内容就是各种Log 
event的集合。



最初，二进制日志是使用基于语句的日志记录编写的。在MySQL 5.1.5中才开始添加了基于行的日志记录。下面几种事件类型专用于基于行的日志记录：TABLE_MAP_EVENT、WRITE_ROWS_EVENT、UPDATE_ROWS_EVENT、DELETE_ROWS_EVENT。

- **FORMAT_DESCRIPTION_EVENT**

格式描述事件是binlog version 4中为了取代之前版本中的START_EVENT_3事件而引入的。它是所有binlog文件中的第一个事件，该事件在一个binlog文件中仅会出现一次。MySQL根据FORMAT_DESCRIPTION_EVENT事件的定义来解析binlog中的其他事件。

FORMAT_DESCRIPTION_EVENT由通用事件头和事件体组成，事件体各字段具体含义如下：

| 字段                        | 长度   | 位置         | 说明                                       |
| ------------------------- | ---- | ---------- | ---------------------------------------- |
| binlog-version            | 2字节  | event-body | binlog版本                                 |
| mysql-server version      | 50字节 | event-body | 服务器版本                                    |
| create-timestamp          | 4字节  | event-body | 该字段指明该binlog文件的创建时间。如果该binlog是由于切换而产生的（指flush logs命令或者binlog文件的大小达到max_binlog_size参数指定的值），那么将该字段设置为0 |
| event header length       | 1字节  | event-body | 19                                       |
| event type header lengths | 数组   | event-body | 该字段是一个数组，记录了所有事件的私有事件头的长度                |

- **QUERY_EVENT**

QUERY_EVENT以文本的形式来记录信息。当binlog的格式是statement的时候，执行的SQL语句都记录在QUERY_EVENT中。

```
+------------------+------+----------------+-----------+-------------+----------------------------------------------+
| mysql-bin.000006 | 1221 | Query          |       200 |        1291 | BEGIN                                        |
| mysql-bin.000006 | 1476 | Query          |       200 |        1580 | use `db`; alter table t1 change id id bigint |
+------------------+------+----------------+-----------+-------------+----------------------------------------------+
```

QUERY_EVENT由通用事件头、私有事件头和事件体3部分组成：

| 字段                 | 长度                 | 位置          | 说明                                       |
| ------------------ | ------------------ | ----------- | ---------------------------------------- |
| slave-proxy-id     | 4字节                | post-header | 某些查询可能会创建临时表，而这些临时表仅仅在当前的连接或会话中有效。为了区分不同连接或会话中的临时表，slave_proxy_id存储了不同连接或会话的线程id |
| execution time     | 4字节                | post-header | 查询从开始执行到记录到binlog所花的时间，单位为秒              |
| schema length      | 1字节                | post-header | schema字符串长度                              |
| error-code         | 2字节                | post-header | 错误码                                      |
| status-vars length | 2字节                | post-header | status-vars长度                            |
| status-vars        | status-vars length | event-body  | status-vars字段是以键值对的形式保存起来的一系列由SET命令设置的上下文信息，例如是否开启autocommit |
| schema             | schema length      | event-body  | 当前选择数据库                                  |
| query              | 取决于查询的长度           | event-body  | query的文本格式，里面存储的可能是BEGIN、COMMIT字符串或者原生的SQL语句，等等 |

QUERY_EVENT类型的事件通常在以下几种情况中使用：

\1. 事务开始时，在binlog中记录一个类型为QUERY_EVENT的BEGIN事件。

\2. 在STATEMENT格式的binlog中，具体执行的SQL语句保存在QUERY_EVENT事件中。

\3. 对于ROW格式的binlog，所有的DDL操作以文本的格式记录在QUERY_EVENT事件中。

- **TABLE_MAP_EVENT**

在ROW格式的binlog文件中，每个ROWS_EVENT（包括：WRITE_ROWS_EVENT、UPDATE_ROWS_EVENT、DELETE_ROWS_EVENT）事件之前都有一个TABLE_MAP_EVENT，用于描述表的内部id和结构定义。也是为了保证slave正确复制数据的重要event。

```
+------------------+------+----------------+-----------+-------------+-------------------------------------------------+
| mysql-bin.000006 |  907 | Anonymous_Gtid |       200 |         972 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'            |
| mysql-bin.000006 |  972 | Query          |       200 |        1042 | BEGIN                                           |
| mysql-bin.000006 | 1042 | Table_map      |       200 |        1085 | table_id: 336 (db.t1)                           |
| mysql-bin.000006 | 1085 | Write_rows     |       200 |        1125 | table_id: 336 flags: STMT_END_F                 |
| mysql-bin.000006 | 1125 | Xid            |       200 |        1156 | COMMIT /* xid=320 */                            |
+------------------+------+----------------+-----------+-------------+-------------------------------------------------+
```



TABLE_MAP_EVENT各个字段的含义如下：

| 字段              | 长度                 | 位置          | 说明                |
| --------------- | ------------------ | ----------- | ----------------- |
| table-id        | 4-6字节              | post-header | 表id               |
| flags           | 2字节                | post-header | 标志位，暂时未使用         |
| schema name     | schema name length | event-body  | 表所在数据库的名称         |
| table name      | table name length  | event-body  | 表名                |
| column-count    | 1、3、4或9字节          | event-body  | 表的列数              |
| column-type-def | column-count       | event-body  | 列的类型              |
| column-meta-def | 长度取决于列的类型          | event-body  | 列的元信息             |
| null-bitmap     | (column-count+7)/8 | event-body  | 以位图的形式记录可以为NULL的列 |

- **WRITE_ROWS_EVENT/UPDATE_ROWS_EVENT/DELETE_ROWS_EVENT**

对于STATEMENT格式的binlog，所有的增删改查操作的原生SQL语句都记录在QUERY_EVENT中，而对于ROW格式的binlog以ROWS_EVENT的形式记录对数据库数据的修改。ROWS_EVENT分为3种：WRITE_ROWS_EVENT、UPDATE_ROWS_EVENT和DELETE_ROWS_EVENT，分别对应于INSERT、UPDATE和DELETE语句。WRITE_ROWS_EVENT包含了要插入的数据；UPDATE_ROWS_EVENT不仅包含了行修改后的值，也包含了行修改前的值；DELETE_ROWS_EVENT仅仅包含了需要删除行的主键值/行号，这也就是为什么表没有主键时会造成[从库延迟](http://www.ywnds.com/%20http://www.ywnds.com/?p=12775)。

ROWS_EVENT各个字段的含义，如下：

| 字段                      | 长度                             | 位置          | 说明                                       |
| ----------------------- | ------------------------------ | ----------- | ---------------------------------------- |
| table-id                | 4-6字节                          | post-header | 该ROWS_EVENT对应的表id                        |
| flags                   | 2字节                            | post-header | 可以包含以下信息：该ROWS_EVENT是否是语句的最后一个事件，是否需要进行外键约束的检查，针对InnoDB的二级索引是否需要进行唯一性检查，该ROWS_EVENT是否包含了完整一行的数据，也就是说覆盖了所有列 |
| extra-data-len          | 2字节                            | post-header | 表所在数据库的名称                                |
| extra-data              | extra-data-len                 | post-header | extra-data的长度                            |
| number of columns       | 1、3、4或9字节                      | event-body  | 仅在版本2的ROWS_EVENT中存在，用于携带额外的数据，主要目的是用于扩展  |
| columns-present-bitmap1 | (column-count+7)/8             | event-body  | 以位图的形式指示了该ROWS_EVENT包含了哪些列的数据            |
| columns-present-bitmap2 | (column-count+7)/8             | event-body  | 对于新版的UPDATE_ROWS_EVENT事件，不仅包含列修改后的值，还包含列修改前的值 |
| null-bitmap             | (bit set in(column-count+7)/8) | event-body  | columns-present-bitmap中为空的列，会以NULL或者列对应的默认值补全 |
| value of columns        | 取决于列的值                         | event-body  | 列的数据                                     |

- **XID_EVENT**

当事务提交时，不管是STATEMENT还是ROW格式的binlog，都会添加一个XID_EVENT事件作为事务的结束。该事件记录了该事务的id。在MySQL进行崩溃恢复的时候，根据事务在binlog中的提交情况来决定是否提交存储引擎中状态为prepared的事务。

```
+------------------+------+----------------+-----------+-------------+----------------------------------------------+
| mysql-bin.000006 | 1380 | Xid            |       200 |        1411 | COMMIT /* xid=326 */                         |
+------------------+------+----------------+-----------+-------------+----------------------------------------------+
```



- **PREVIOUS_GTIDS_LOG_EVENT/GTID_LOG_EVENT**

MySQL 5.6引入全局事务ID的首要目的，是保证Slave在复制的时候不会重复执行相同的事务操作；其次，是用全局事务IDs代替由文件名和物理偏移量组成的复制位点，定位Slave需要复制的binlog内容。因此，MySQL必须在写binlog时记录每个事务的全局GTID，保证Master / Slave可以根据这些GTID忽略或者执行相应的事务。在实现上，MySQL没有修改旧的binlog事件，而是新增了两类事件：

PREVIOUS_GTIDS_LOG_EVENT：用于表示上一个binlog最后一个gitd的位置，每个binlog只有一个，当没有开启GTID时此事件为空。

GTID_LOG_EVENT：当开启GTID时，每一个操作语句（DML/DDL）执行前就会添加一个GTID事件，记录当前全局事务ID；同时在MySQL 5.7版本中，组提交信息也存放在GTID事件中，有两个关键字段last_committed，sequence_number就是用来标识组提交信息的。

下面一个新的binlog文件在开启GTID后执行一个DML语句产生的事件：

```
+------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------+
| mysql-bin.000008 |   4 | Format_desc    |       200 |         123 | Server ver: 5.7.18-log, Binlog ver: 4                             |
| mysql-bin.000008 | 123 | Previous_gtids |       200 |         194 | 8ae691e7-33d8-11e7-be18-000c2916018b:1-2                          |
| mysql-bin.000008 | 194 | Gtid           |       200 |         259 | SET @@SESSION.GTID_NEXT= '8ae691e7-33d8-11e7-be18-000c2916018b:3' |
| mysql-bin.000008 | 259 | Query          |       200 |         329 | BEGIN                                                             |
| mysql-bin.000008 | 329 | Table_map      |       200 |         372 | table_id: 155 (db.t1)                                             |
| mysql-bin.000008 | 372 | Write_rows     |       200 |         416 | table_id: 155 flags: STMT_END_F                                   |
| mysql-bin.000008 | 416 | Xid            |       200 |         447 | COMMIT /* xid=34 */                                               |
+------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------+
```



来看一下对应mysqlbinlog -vv解析的信息：

```
# at 123
#180123 22:48:05 server id 200  end_log_pos 194 CRC32 0xde76092d        Previous-GTIDs
# 8ae691e7-33d8-11e7-be18-000c2916018b:1-2
# at 194
#180123 22:48:08 server id 200  end_log_pos 259 CRC32 0xacaf9041        GTID    last_committed=0        sequence_number=1
SET @@SESSION.GTID_NEXT= '8ae691e7-33d8-11e7-be18-000c2916018b:3'/*!*/;
# at 259
#180123 22:48:08 server id 200  end_log_pos 329 CRC32 0x599d5c6c        Query   thread_id=4     exec_time=0     error_code=0
.....
```

关键看Previous_gtids与Gtid事件，相关信息与我们表述的基本一致。

- **ANONYMOUS_GTID_LOG_EVENT**

这个事件是在MySQL 5.7版本中新增的，在5.7版本中MySQL为了识别事务是否在同一个组中，就将组提交（Group Commit）的信息存放在GTID中。那么如果用户没有开启GTID功能，即将参数gtid_mode设置为OFF呢？故MySQL 5.7又引入了称之为Anonymous_Gtid（ANONYMOUS_GTID_LOG_EVENT）的二进制日志event类型。这意味着在MySQL 5.7版本中即使不开启GTID，每个事务开始前也是会存在一个Anonymous_Gtid，而这个Anonymous_Gtid事件中就存在着组提交的信息。反之，如果开启了GTID后，就不会存在这个Anonymous_Gtid了，从而组提交信息就记录在非匿名GTID事件中。

通过mysqlbinlog工具可以看到这个事件，有两个关键字段last_committed，sequence_number就是用来标识组提交信息的，如下：

```

# at 1752
#180123  5:35:21 server id 200  end_log_pos 1817 CRC32 0x10ed8d0c       Anonymous_GTID  last_committed=7        sequence_number=8
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
```

- **ROTATE_EVENT**

当binlog文件的大小达到max_binlog_size参数设置的值时或者执行flush logs命令时，binlog会发送切换，这时会在当前使用的binlog文件末尾添加一个ROTATE_EVENT事件，将下一个binlog文件的名称记录在该事件中。

```
+------------------+-----+----------------+-----------+-------------+---------------------------------------+
| Log_name         | Pos | Event_type     | Server_id | End_log_pos | Info                                  |
+------------------+-----+----------------+-----------+-------------+---------------------------------------+
| mysql-bin.000009 | 312 | Rotate         |       200 |         359 | mysql-bin.000006;pos=4                |
+------------------+-----+----------------+-----------+-------------+---------------------------------------+
```

- **STOP_EVENT**

当MySQL停止时，会在当前binlog文件的结尾写入一个STOP_EVENT事件来表示数据库停止。STOP_EVENT仅仅包含一个通用事件头，没有私有事件头和事件体部分，因为只需要在公有事件头的type code字段指定为STOP_EVENT就可以了，不需要携带额外的信息。

```
+------------------+-----+----------------+-----------+-------------+---------------------------------------+
| Log_name         | Pos | Event_type     | Server_id | End_log_pos | Info                                  |
+------------------+-----+----------------+-----------+-------------+---------------------------------------+
| mysql-bin.000009 | 405 | Stop           |       200 |         428 |                                       |
+------------------+-----+----------------+-----------+-------------+---------------------------------------+
```

- **BINLOG_CHECKPOINT_EVENT**

该事件是MariaDB引入的新事件，主要用于崩溃恢复。在两阶段事务提交过程中，当MariaDB崩溃时，我们需要根据binlog中事务的提交情况来决定是否提交存储引擎内部状态为prepared的事务。为了减少恢复过程中需要读取的binlog文件数，当某个binlog文件内部的所有事务都在存储引擎内部提交了，这时我们会在binlog中写入一个BINLOG_CHECKPOINT_EVENT事件。执行崩溃恢复的过程中，MariaDB会根据读取的BINLOG_CHECKPOINT_EVENT来决定哪些binlog文件是可以不用扫描的。

```
+------------------+-----+-------------------+-----------+-------------+--------------------------------------------+
| Log_name         | Pos | Event_type        | Server_id | End_log_pos | Info                                       |
+------------------+-----+-------------------+-----------+-------------+--------------------------------------------+
| mysql-bin.000010 | 307 | Binlog_checkpoint |       300 |         350 | mysql-bin.000009                           |
+------------------+-----+-------------------+-----------+-------------+--------------------------------------------+
```

# 参考资料

http://www.ywnds.com/?p=12839