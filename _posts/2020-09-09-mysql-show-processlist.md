---
layout: post
title: show processlist命令
date: 2020-09-09
categories:
    - MySQL
comments: true
permalink: show-processlist.html
---

`show processlist` 或 `show full processlist` 用来显示用户正在运行的线程。它返回的结果是实时变化的，是对 mysql 链接执行的现场快照，所以用来处理突发事件非常有用。

它可以查看当前 mysql 的一些运行情况，是否有压力，都在执行什么 sql，语句耗时几何，有没有慢 sql 在执行等等。

需要注意的是，除了 root 用户能看到所有正在运行的线程外，其他用户都只能看到自己正在运行的线程，看不到其它用户正在运行的线程。除非单独个这个用户赋予了PROCESS 权限。

`show processlist` 显示的信息都是来自MySQL系统库 `information_schema` 中的 `processlist` 表。所以使用下面的查询语句可以获得相同的结果：

```
select * from information_schema.processlist
```

> 有时候一个快照可能看不出什么问题，那么可以频发的刷新试试

```
mysql> show full processlist;
+--------+------+----------------------+-------+---------+------+----------+-----------------------+
| Id     | User | Host                 | db    | Command | Time | State    | Info                  |
+--------+------+----------------------+-------+---------+------+----------+-----------------------+
| 449000 | root | 127.123.213.11:59828 | stark | Sleep   | 1270 |          | NULL                  |
| 449001 | root | 127.123.213.11:59900 | stark | Sleep   | 1241 |          | NULL                  |
| 449002 | root | 127.123.213.11:59958 | stark | Sleep   | 1216 |          | NULL                  |
| 449003 | root | 127.123.213.11:60088 | stark | Sleep   | 1159 |          | NULL                  |
| 449004 | root | 127.123.213.11:60108 | stark | Sleep   | 1151 |          | NULL                  |
| 449005 | root | 127.123.213.11:60280 | stark | Sleep   | 1076 |          | NULL                  |
| 449006 | root | 127.123.213.11:60286 | stark | Sleep   | 1074 |          | NULL                  |
| 449007 | root | 127.123.213.11:60344 | stark | Sleep   | 1052 |          | NULL                  |
| 449008 | root | 127.123.213.11:60450 | stark | Sleep   | 1005 |          | NULL                  |
| 449009 | root | 127.123.213.11:60498 | stark | Sleep   |  986 |          | NULL                  |
| 449013 | root | localhost            | NULL  | Query   |    0 | starting | show full processlist |
+--------+------+----------------------+-------+---------+------+----------+-----------------------+
11 rows in set (0.01 sec)

mysql> show full processlist\G;
*************************** 1. row ***************************
     Id: 449000
   User: root
   Host: 127.123.213.11:59828
     db: stark
Command: Sleep
   Time: 1283
  State: 
   Info: NULL
*************************** 2. row ***************************
     Id: 449001
   User: root
   Host: 127.123.213.11:59900
     db: stark
Command: Sleep
   Time: 1254
  State: 
   Info: NULL

```

# 1. 各列说明

先简单说一下各列的含义和用途，

- Id: 线程ID，当我们发现这个线程有问题的时候，可以通过 kill 命令，加上这个Id值将这个线程杀掉。这个ID就是`information_schema.processlist`表的主键
- User: 就是指启动这个线程的用户。
- Host: 记录了发送请求的客户端的 IP 和 端口号。通过这些信息在排查问题的时候，我们可以定位到是哪个客户端的哪个进程发送的请求。
- DB: 当前执行的命令是在哪一个数据库上。如果没有指定数据库，则该值为 NULL 。
- Command: 是指此刻该线程正在执行的命令。
- Time: 表示该线程处于当前状态的时间，单位是秒。
- State: 线程的状态，和 Command 对应。请注意，state只是语句执行中的某一个状态，一个sql语句，已查询为例，可能需要经过copying to tmp table，Sorting result，Sending data等状态才可以完成。
- Info: 一般记录的是线程执行的语句。默认只显示前100个字符，也就是你看到的语句可能是截断了的，要看全部信息，需要使用 show full processlist。

## 1.1. Command列：


- Binlog Dump: 主节点正在将二进制日志 ，同步到从节点
- Change User: 正在执行一个 change-user 的操作
- Close Stmt: 正在关闭一个Prepared Statement 对象
- Connect: 一个从节点连上了主节点
- Connect Out: 一个从节点正在连主节点
- Create DB: 正在执行一个create-database 的操作
- Daemon: 服务器内部线程，而不是来自客户端的链接
- Debug: 线程正在生成调试信息
- Delayed Insert: 该线程是一个延迟插入的处理程序
- Drop DB: 正在执行一个 drop-database 的操作
- Execute: 正在执行一个 Prepared Statement 
- Fetch: 正在从Prepared Statement 中获取执行结果
- Field List: 正在获取表的列信息
- Init DB: 该线程正在选取一个默认的数据库
- Kill : 正在执行 kill 语句，杀死指定线程
- Long Data: 正在从Prepared Statement 中检索 long data
- Ping: 正在处理 server-ping 的请求
- Prepare: 该线程正在准备一个 Prepared Statement
- ProcessList: 该线程正在生成服务器线程相关信息
- Query: 该线程正在执行一个语句
- Quit: 该线程正在退出
- Refresh：该线程正在刷表，日志或缓存；或者在重置状态变量，或者在复制服务器信息
- Register Slave： 正在注册从节点
- Reset Stmt: 正在重置 prepared statement
- Set Option: 正在设置或重置客户端的 statement-execution 选项
- Shutdown: 正在关闭服务器
- Sleep: 正在等待客户端向它发送执行语句
- Statistics: 该线程正在生成 server-status 信息
- Table Dump: 正在发送表的内容到从服务器
- Time: Unused

## 1.2. state列

- Checking table 	正在检查数据表（这是自动的）。
- Closing tables 	正在将表中修改的数据刷新到磁盘中，同时正在关闭已经用完的表。这是一个很快的操作，如果不是这样的话，就应该确认磁盘空间是否已经满了或者磁盘是否正处于重负中。
- Connect Out 	复制从服务器正在连接主服务器。
- Copying to tmp table on disk 	由于临时结果集大于tmp_table_size，正在将临时表从内存存储转为磁盘存储以此节省内存。
- Creating tmp table 	正在创建临时表以存放部分查询结果。
- deleting from main table 	服务器正在执行多表删除中的第一部分，刚删除第一个表。
- deleting from reference tables 	服务器正在执行多表删除中的第二部分，正在删除其他表的记录。
- Flushing tables 	正在执行FLUSH TABLES，等待其他线程关闭数据表。
- Killed 	发送了一个kill请求给某线程，那么这个线程将会检查kill标志位，同时会放弃下一个kill请求。MySQL会在每次的主循环中检查kill标志位，不过有些情况下该线程可能会过一小段才能死掉。如果该线程程被其他线程锁住了，那么kill请求会在锁释放时马上生效。
- Locked 	被其他查询锁住了。
- Sending data 	正在处理SELECT查询的记录，同时正在把结果发送给客户端。
- Sorting for group 	正在为GROUP BY做排序。
- Sorting for order 	正在为ORDER BY做排序。
- Opening tables 	这个过程应该会很快，除非受到其他因素的干扰。例如，在执ALTER TABLE或LOCK TABLE语句行完以前，数据表无法被其他线程打开。正尝试打开一个表。
- Removing duplicates 	正在执行一个SELECT DISTINCT方式的查询，但是MySQL无法在前一个阶段优化掉那些重复的记录。因此，MySQL需要再次去掉重复的记录，然后再把结果发送给客户端。
- Reopen table 	获得了对一个表的锁，但是必须在表结构修改之后才能获得这个锁。已经释放锁，关闭数据表，正尝试重新打开数据表。
- Repair by sorting 	修复指令正在排序以创建索引。
- Repair with keycache 	修复指令正在利用索引缓存一个一个地创建新索引。它会比Repair by sorting慢些。
- Searching rows for update 	正在讲符合条件的记录找出来以备更新。它必须在UPDATE要修改相关的记录之前就完成了。
- Sleeping 	正在等待客户端发送新请求.
- System lock 	正在等待取得一个外部的系统锁。如果当前没有运行多个mysqld服务器同时请求同一个表，那么可以通过增加--skip-external-locking参数来禁止外部系统锁。
- Upgrading lock 	INSERT DELAYED正在尝试取得一个锁表以插入新记录。
- Updating 	正在搜索匹配的记录，并且修改它们。
- User Lock 	正在等待GET_LOCK()。
- Waiting for tables 	该线程得到通知，数据表结构已经被修改了，需要重新打开数据表以取得新的结构。然后，为了能的重新打开数据表，必须等到所有其他线程关闭这个表。以下几种情况下会产生这个通知：FLUSH TABLES tbl_name, ALTER TABLE, RENAME TABLE, REPAIR TABLE, ANALYZE TABLE,或OPTIMIZE TABLE。
- waiting for handler insert 	INSERT DELAYED已经处理完了所有待处理的插入操作，正在等待新的请求。

# 2. 常用查询
## 2.1. 根据某个用户过滤

```
select * from information_schema.processlist where User='UserName';
```

## 2.2. 杀死某些线程
```
select concat('kill ',ID,';') from information_schema.processlist where User='UserName';
```

批量结束超出3分钟的线程

```
select concat('kill ', id, ';')
from information_schema.processlist
where command != 'Sleep'
and time > 3*60
order by time desc;
```

## 2.3. 监控统计每个用户的访问

```
select User,count(*) as cnt from information_schema.processlist group by user;
```


## 2.4. 监控统计每个Ip的访问

```
select client_ip,count(client_ip) as client_num 
from (select substring_index(host,':' ,1) as client_ip from processlist ) as connect_info 
group by client_ip 
order by client_num desc;
```

## 2.5 查看正在执行的线程，并按 Time 倒排序

```
select * from information_schema.processlist where Command != 'Sleep' order by Time desc;
```

## 2.6. 查看死锁

1. 查出死锁的线程(state为“locked”的状)
```
mysql -u root -e "show processlist" | grep -i "Locked" >> locked_log.txt
```

2. 生成kill

```
for line in `cat locked_log.txt | awk '{print $1}'`
do
   echo "kill $line;" >> kill_thread_id.sql
done
```

# 3. 参考资料

https://developer.aliyun.com/article/585738

https://zhuanlan.zhihu.com/p/30743094
