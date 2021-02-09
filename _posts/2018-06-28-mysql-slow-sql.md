---
layout: post
title: MySQL日志（3）-慢查询日志
date: 2019-06-28
categories:
    - MySQL
comments: true
permalink: mysql-slow-sql.html
---

MySQL的慢查询日志是MySQL提供的一种日志记录，它用来记录在MySQL中响应时间超过阀值的语句

具体指运行时间超过long_query_time值的SQL，则会被记录到慢查询日志中。long_query_time的默认值为10，意思是运行10S以上的语句。


默认情况下，MySQL数据库并不启动慢查询日志，需要我们手动来设置这个参数，当然，如果不是调优需要的话，一般不建议启动该参数，因为开启慢查询日志会或多或少带来一定的性能影响。慢查询日志支持将日志记录写入文件，也支持将日志记录写入数据库表。

# 1. 参数

- slow_query_log ：是否开启慢查询日志，1表示开启，0表示关闭。    
- log-slow-queries：旧版（5.6以下版本）MySQL数据库慢查询日志存储路径。可以不设置该参数，系统则会默认给一个缺省的文件host_name-slow.log    
- slow-query-log-file：新版（5.6及以上版本）MySQL数据库慢查询日志存储路径。可以不设置该参数，系统则会默认给一个缺省的文件host_name-slow.log    
- long_query_time：慢查询阈值，当查询时间多于设定的阈值时，记录日志。 
- min_examined_row_limit：设置检查的行数大于等于多少行时记录到日志中   
- log_queries_not_using_indexes：未使用索引的查询或全表扫描的 SQL也被记录到慢查询日志中（可选项）。   
- log_throttle_queries_not_using_indexes：开启 log_queries_not_using_indexes 后，此参数会限制每分钟可以写入慢速查询日志的此类查询的数量，参数设置 0 为不限制
- log_output：日志存储方式。

  - log_output='FILE'表示将日志存入文件，默认值是'FILE'。
  - log_output='TABLE'表示将日志存入数据库，这样日志信息就会被写入到mysql.slow_log表中。

MySQL数据库支持同时两种日志存储方式，配置的时候以逗号隔开即可，如：`log_output='FILE,TABLE'`。

日志记录到系统的专用日志表中，要比记录到文件耗费更多的系统资源，因此对于需要启用慢查询日志，又需要能够获得更高的系统性能，那么建议优先记录到文件。

```
mysql> show variables like '%slow_query_log%';
+---------------------+--------------------------------------+
| Variable_name       | Value                                |
+---------------------+--------------------------------------+
| slow_query_log      | OFF                                  |
| slow_query_log_file | /var/lib/mysql/9849434cd7bb-slow.log |
+---------------------+--------------------------------------+
2 rows in set (0.00 sec)
```
## 1.1. 开启慢查询日志

```
mysql> set global slow_query_log=1;
Query OK, 0 rows affected (0.01 sec)

mysql> show variables like '%slow_query_log%';
+---------------------+--------------------------------------+
| Variable_name       | Value                                |
+---------------------+--------------------------------------+
| slow_query_log      | ON                                   |
| slow_query_log_file | /var/lib/mysql/9849434cd7bb-slow.log |
+---------------------+--------------------------------------+
2 rows in set (0.00 sec)
```
**也可以直接修改mysql配置文件**
使用`set global slow_query_log=1`开启了慢查询日志只对当前数据库生效，如果MySQL重启后则会失效。如果要永久生效，就必须修改配置文件my.cnf（其它系统变量也是如此）。

```
slow_query_log =1
slow_query_log_file=/tmp/mysql_slow.log
```

## 1.2. long_query_time

那么开启了慢查询日志后，什么样的SQL才会记录到慢查询日志里面呢？ 

这个是由参数long_query_time控制，默认情况下long_query_time的值为10秒，可以使用命令修改，也可以在my.cnf参数里面修改。运行时间正好等于long_query_time的情况，并不会被记录下来。也就是说，在mysql源码里是判断大于long_query_time，而非大于等于。

从MySQL 5.1开始，long_query_time开始以微秒记录SQL语句运行时间，之前仅用秒为单位记录。如果记录到表里面，只会记录整数部分，不会记录微秒部分。

```
mysql> show variables like '%long_query_time%';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+
1 row in set (0.01 sec)

mysql> set global long_query_time=1;
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like '%long_query_time%';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+
1 row in set (0.00 sec)
```

我修改了变量long_query_time，但是查询变量long_query_time的值还是10，难道没有修改到呢？注意：使用命令 `set global long_query_time=4`修改后，需要重新连接或新开一个会话才能看到修改值。你用`show variables like 'long_query_time'`查看是当前会话的变量值，你也可以不用重新连接会话，而是用`show global variables like 'long_query_time'`; 

```
mysql> show global variables like '%long_query_time%';
+-----------------+----------+
| Variable_name   | Value    |
+-----------------+----------+
| long_query_time | 1.000000 |
+-----------------+----------+
1 row in set (0.00 sec)
```
## 1.3. log_output
log_output 参数是指定日志的存储方式。

- log_output='FILE'表示将日志存入文件，默认值是'FILE'。
- 
log_output='TABLE'表示将日志存入数据库，这样日志信息就会被写入到mysql.slow_log表中。

MySQL数据库支持同时两种日志存储方式，配置的时候以逗号隔开即可，如：log_output='FILE,TABLE'。

日志记录到系统的专用日志表中，要比记录到文件耗费更多的系统资源，因此对于需要启用慢查询日志，又需要能够获得更高的系统性能，那么建议优先记录到文件。
```
mysql> show variables like '%log_output%';     
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_output    | FILE  |
+---------------+-------+
1 row in set (0.00 sec)
```

## 1.4. log-queries-not-using-indexes

系统变量log-queries-not-using-indexes：未使用索引的查询也被记录到慢查询日志中（可选项）。如果调优的话，建议开启这个选项。另外，开启了这个参数，其实使用full index scan的sql也会被记录到慢查询日志。

> This option does not necessarily mean that no index is used. For example, a query that uses a full index scan uses an index but would be logged because the index would not limit the number of rows.

```	
mysql> show variables like '%log_queries_not_using_indexes%';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| log_queries_not_using_indexes | OFF   |
+-------------------------------+-------+
1 row in set (0.00 sec)

mysql> set global log_queries_not_using_indexes=1;
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like '%log_queries_not_using_indexes%';                                                      
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| log_queries_not_using_indexes | ON    |
+-------------------------------+-------+
1 row in set (0.00 sec)
```

## 1.5. log_slow_admin_statements

系统变量log_slow_admin_statements表示是否将慢管理语句例如ANALYZE TABLE和ALTER TABLE等记入慢查询日志。

```
mysql> show variables like '%log_slow_admin_statements%';    
+---------------------------+-------+
| Variable_name             | Value |
+---------------------------+-------+
| log_slow_admin_statements | OFF   |
+---------------------------+-------+
1 row in set (0.00 sec)
```

## 1.6. slow_queries

如果你想查询有多少条慢查询记录，可以使用系统变量。

```
mysql> show global status like '%slow_queries%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Slow_queries  | 823   |
+---------------+-------+
1 row in set (0.00 sec)
```

# 2.  less 或 more 命令来查看

对于我们详细来分析 SQL 的话，一般采用这种方式，查找到对应时间点的对应 SQL 来进行分析。

```
show master status;
# Time: 2020-11-16T08:27:16.777259+08:00
# User@Host: root[root] @ [127.0.0.1] Id: 248
# Query_time: 15.293745 Lock_time: 0.000000 Rows_sent: 0 Rows_examined: 0
SET timestamp=1605486436;
```

那么如何读懂慢日志里面对这些信息呢？

```
show master status 	#慢 SQL
Time 		        #出现该慢 SQL 的时间
query_time			# SQL 语句的查询时间(在 MySQL 中所有类型的 SQL 语句执行的时间都叫做 query_time,而在 Oracle 中则仅指 select)
lock_time: 	        #锁的时间
rows_sent: 			#返回了多少行,如果做了聚合就不准确了
rows_examined:		#执行这条 SQL 处理了多少行数据
SET timestamp 		#时间戳
```

通过这些我们就可以来明确的知道一条 SQL 究竟执行了多长时间的查询，有没有发生锁等待，此查询实际在数据库中读取了多少行数据了。

# 3. mysqldumpslow    

```                                     
Option h requires an argument
ERROR: bad option

Usage: mysqldumpslow [ OPTS... ] [ LOGS... ]

Parse and summarize the MySQL slow query log. Options are

  --verbose    verbose
  --debug      debug
  --help       write this text to standard output

  -v           verbose
  -d           debug
  -s ORDER     what to sort by (al, at, ar, c, l, r, t), 'at' is default
				al: average lock time
				ar: average rows sent
				at: average query time
				 c: count
				 l: lock time
				 r: rows sent
				 t: query time  
  -r           reverse the sort order (largest last instead of first)
  -t NUM       just show the top n queries
  -a           don't abstract all numbers to N and strings to 'S'
  -n NUM       abstract numbers with at least n digits within names
  -g PATTERN   grep: only consider stmts that include this string
  -h HOSTNAME  hostname of db server for *-slow.log filename (can be wildcard),
			   default is '*', i.e. match all
  -i NAME      name of server instance (if using mysql.server startup script)
  -l           don't subtract lock time from total time
```

说明 

```
-s, 是表示按照何种方式排序；
    c: 访问计数    
    l: 锁定时间    
    r: 返回记录    
    t: 查询时间    
    al:平均锁定时间    
    ar:平均返回记录数    
    at:平均查询时间
-t, 是top n的意思，即为返回前面多少条的数据；
-g, 后边可以写一个正则匹配模式，大小写不敏感的；
```
**得到返回记录集最多的10个SQL**

```
mysqldumpslow -s r -t 10 9849434cd7bb-slow.log
```
**得到访问次数最多的1个SQL**

```
root@9849434cd7bb:/var/lib/mysql# mysqldumpslow -s c -t 1 9849434cd7bb-slow.log  

Reading mysql slow query log from 9849434cd7bb-slow.log
Count: 1272  Time=0.00s (0s)  Lock=0.00s (0s)  Rows=5.0 (6360), admin[admin]@[192.168.0.1]
    select
    company_id, company_code, name, address, state, app_key, app_secret, company_key, level, scope
    from company
    WHERE  state = N 
    limit
    N,
    N
```
**得到按照时间排序的前10条里面含有左连接的查询语句。**

```
mysqldumpslow -s t -t 10 -g “left join” /database/mysql/mysql06_slow.log
```

**另外建议在使用这些命令时结合 | 和more 使用 ，否则有可能出现刷屏的情况。**

```
mysqldumpslow -s c -t 10 9849434cd7bb-slow.log | more 
```

# 4. pt-query-digest

pt-query-digest 属于 Percona Toolkit 工具集中较为常用的工具，用于分析 slow log，可以分析 MySQL  数据库的 binary log 、 general log 日志，同时也可以使用 show processlist 或从 tcpdump 抓取的 MySQL 协议数据来进行分析。

**语法**

```html
pt-query-digest [OPTIONS] [FILES] [DSN]

--create-review-table  当使用--review参数把分析结果输出到表中时，如果没有表就自动创建。
--create-history-table  当使用--history参数把分析结果输出到表中时，如果没有表就自动创建。
--filter  对输入的慢查询按指定的字符串进行匹配过滤后再进行分析
--limit    限制输出结果百分比或数量，默认值是20,即将最慢的20条语句输出，如果是50%则按总响应时间占比从大到小排序，输出到总和达到50%位置截止。
--host  mysql服务器地址
--user  mysql用户名
--password  mysql用户密码
--history 将分析结果保存到表中，分析结果比较详细，下次再使用--history时，如果存在相同的语句，且查询所在的时间区间和历史表中的不同，则会记录到数据表中，可以通过查询同一CHECKSUM来比较某类型查询的历史变化。
--review 将分析结果保存到表中，这个分析只是对查询条件进行参数化，一个类型的查询一条记录，比较简单。当下次使用--review时，如果存在相同的语句分析，就不会记录到数据表中。
--output 分析结果输出类型，值可以是report(标准分析报告)、slowlog(Mysql slow log)、json、json-anon，一般使用report，以便于阅读。
--since 从什么时间开始分析，值为字符串，可以是指定的某个”yyyy-mm-dd [hh:mm:ss]”格式的时间点，也可以是简单的一个时间值：s(秒)、h(小时)、m(分钟)、d(天)，如12h就表示从12小时前开始统计。
--until 截止时间，配合—since可以分析一段时间内的慢查询。
```

```
# pt-query-digest   mysql-slow.log

# 3.6s user time, 100ms system time, 32.64M rss, 227.52M vsz
# Current date: Wed Sep  2 11:22:34 2020
# Hostname: xxx
# Files: mysql-slow.log
# Overall: 37.46k total, 17 unique, 576.26 QPS, 5.58x concurrency ________
# Time range: 2020-09-02T11:21:24 to 2020-09-02T11:22:29
# Attribute          total     min     max     avg     95%  stddev  median
# ============     ======= ======= ======= ======= ======= ======= =======
# Exec time           363s     1ms   183ms    10ms    23ms     8ms     8ms
# Lock time             1s       0    34ms    35us    38us   603us       0
# Rows sent        182.14k       0     100    4.98    0.99   21.06       0
# Rows examine     491.83k       0   1.02k   13.45   97.36   56.85       0
# Query size        19.82M       5 511.96k  554.80   72.65  16.25k    5.75

# Profile
# Rank Query ID                      Response time  Calls R/Call V/M   Ite
# ==== ============================= ============== ===== ====== ===== ===
#    1 0xFFFCA4D67EA0A788813031B8... 328.2315 90.4% 30520 0.0108  0.01 COMMIT
#    2 0xB2249CB854EE3C2AD30AD7E3...   8.0186  2.2%  1208 0.0066  0.01 UPDATE sbtest?
#    3 0xE81D0B3DB4FB31BC558CAEF5...   6.6346  1.8%  1639 0.0040  0.01 SELECT sbtest?
#    4 0xDDBF88031795EC65EAB8A8A8...   5.5093  1.5%   756 0.0073  0.02 DELETE sbtest?
# MISC 0xMISC                         14.6011  4.0%  3334 0.0044   0.0 <13 ITEMS>

# Query 1: 1.02k QPS, 10.94x concurrency, ID 0xFFFCA4D67EA0A788813031B8BBC3B329 at byte 26111916
# This item is included in the report because it matches --limit.
# Scores: V/M = 0.01
# Time range: 2020-09-02T11:21:59 to 2020-09-02T11:22:29
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count         81   30520
# Exec time     90    328s     1ms   129ms    11ms    23ms     8ms     9ms
# Lock time      0       0       0       0       0       0       0       0
# Rows sent      0       0       0       0       0       0       0       0
# Rows examine   0       0       0       0       0       0       0       0
# Query size     0 178.83k       6       6       6       6       0       6
# String:
# Databases    test
# Hosts        10.186.60.147
# Users        root
# Query_time distribution
#   1us
#  10us
# 100us
#   1ms  ################################################################
#  10ms  ###########################################################
# 100ms  #
#    1s
#  10s+
COMMIT\G

# Query 2: 41.66 QPS, 0.28x concurrency, ID 0xB2249CB854EE3C2AD30AD7E3079ABCE7 at byte 24161590
# This item is included in the report because it matches --limit.
# Scores: V/M = 0.01
# Time range: 2020-09-02T11:21:59 to 2020-09-02T11:22:28
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count          3    1208
# Exec time      2      8s     1ms   115ms     7ms    24ms     9ms     3ms
# Lock time     38   518ms    17us    34ms   428us    73us     2ms    36us
# Rows sent      0       0       0       0       0       0       0       0
# Rows examine   0   1.18k       1       1       1       1       0       1
# Query size     0  46.01k      39      39      39      39       0      39
# String:
# Databases    test
# Hosts        10.186.60.147
# Users        root
# Query_time distribution
#   1us
#  10us
# 100us
#   1ms  ################################################################
#  10ms  ##############
# 100ms  #
#    1s
#  10s+
# Tables
#    SHOW TABLE STATUS FROM `test` LIKE 'sbtest1'\G
#    SHOW CREATE TABLE `test`.`sbtest1`\G
UPDATE sbtest1 SET k=k+1 WHERE id=50313\G
# Converted for EXPLAIN
# EXPLAIN /*!50100 PARTITIONS*/
select  k=k+1 from sbtest1 where  id=50313\G

# Query 3: 56.52 QPS, 0.23x concurrency, ID 0xE81D0B3DB4FB31BC558CAEF5F387E929 at byte 22020829
# This item is included in the report because it matches --limit.
# Scores: V/M = 0.01
# Time range: 2020-09-02T11:21:59 to 2020-09-02T11:22:28
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count          4    1639
# Exec time      1      7s     1ms    61ms     4ms    14ms     5ms     2ms
# Lock time      3    45ms    11us   958us    27us    44us    30us    23us
# Rows sent      0   1.60k       1       1       1       1       0       1
# Rows examine   0   1.60k       1       1       1       1       0       1
# Query size     0  57.62k      36      36      36      36       0      36
# String:
# Databases    test
# Hosts        10.186.60.147
# Users        root
# Query_time distribution
#   1us
#  10us
# 100us
#   1ms  ################################################################
#  10ms  ######
# 100ms
#    1s
#  10s+
# Tables
#    SHOW TABLE STATUS FROM `test` LIKE 'sbtest1'\G
#    SHOW CREATE TABLE `test`.`sbtest1`\G
# EXPLAIN /*!50100 PARTITIONS*/
SELECT c FROM sbtest1 WHERE id=61690\G

# Query 4: 26.07 QPS, 0.19x concurrency, ID 0xDDBF88031795EC65EAB8A8A8BEEFF705 at byte 21045172
# This item is included in the report because it matches --limit.
# Scores: V/M = 0.02
# Time range: 2020-09-02T11:21:59 to 2020-09-02T11:22:28
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count          2     756
# Exec time      1      6s     1ms   104ms     7ms    26ms    10ms     3ms
# Lock time     18   252ms    13us    19ms   333us    54us     2ms    26us
# Rows sent      0       0       0       0       0       0       0       0
# Rows examine   0     756       1       1       1       1       0       1
# Query size     0  25.10k      34      34      34      34       0      34
# String:
# Databases    test
# Hosts        10.186.60.147
# Users        root
# Query_time distribution
#   1us
#  10us
# 100us
#   1ms  ################################################################
#  10ms  #################
# 100ms  #
#    1s
#  10s+
# Tables
#    SHOW TABLE STATUS FROM `test` LIKE 'sbtest1'\G
#    SHOW CREATE TABLE `test`.`sbtest1`\G
DELETE FROM sbtest1 WHERE id=50296\G
# Converted for EXPLAIN
# EXPLAIN /*!50100 PARTITIONS*/
select * from  sbtest1 WHERE id=50296\G
```

**第一部分：输出结果的总体信息**

```text
# 3.6s user time, 100ms system time, 32.64M rss, 227.52M vsz 
说明：
执行过程中在用户中所花费的所有时间
执行过程中内核空间中所花费的所有时间
pt-query-digest进程所分配的内存大小
pt-query-digest进程所分配的虚拟内存大小

# Current date: Wed Sep  2 11:22:34 2020 
说明：当前日期

# Hostname: xxx
说明：执行pt-query-digest的主机名

# Files: mysql-slow.log
说明：被分析的文件名

# Overall: 37.46k total, 17 unique, 576.26 QPS, 5.58x concurrency ________
说明：语句总数量，唯一语句数量，每秒查询量，查询的并发

# Time range: 2020-09-02T11:21:24 to 2020-09-02T11:22:29
说明：执行过程中日志记录的时间范围

# Attribute          total     min     max     avg     95%  stddev  median
说明：属性           总计     最小值  最大值  平均值   95%  标准差  中位数
# ============     ======= ======= ======= ======= ======= ======= =======

# Exec time           363s     1ms   183ms    10ms    23ms     8ms     8ms
说明：执行时间
# Lock time             1s       0    34ms    35us    38us   603us       0
说明：锁占用时间
# Rows sent        182.14k       0     100    4.98    0.99   21.06       0
说明：发送到客户端的行数
# Rows examine     491.83k       0   1.02k   13.45   97.36   56.85       0
说明：扫描的语句行数
# Query size        19.82M       5 511.96k  554.80   72.65  16.25k    5.75 
说明：查询的字符数
```

**第二部分：输出队列组的统计信息**

```text
# Profile
说明：简况
# Rank Query ID                      Response time  Calls R/Call V/M   Ite
# ==== ============================= ============== ===== ====== ===== ===
#    1 0xFFFCA4D67EA0A788813031B8... 328.2315 90.4% 30520 0.0108  0.01 COMMIT
#    2 0xB2249CB854EE3C2AD30AD7E3...   8.0186  2.2%  1208 0.0066  0.01 UPDATE sbtest?
#    3 0xE81D0B3DB4FB31BC558CAEF5...   6.6346  1.8%  1639 0.0040  0.01 SELECT sbtest?
#    4 0xDDBF88031795EC65EAB8A8A8...   5.5093  1.5%   756 0.0073  0.02 DELETE sbtest?
# MISC 0xMISC        
```

- Rank：所有语句的排名，默认按查询时间降序排列，通过--order-by指定
-  Query ID：语句的ID，（去掉多余空格和文本字符，计算hash值）
-  Response：总的响应时间
-  time：该查询在本次分析中总的时间占比
-  calls：执行次数，即本次分析总共有多少条这种类型的查询语句
-  R/Call：平均每次执行的响应时间
-  V/M：响应时间Variance-to-mean的比率
-  Item：查询对象

# 5. 如何在线安全清空 slow.log 文件

在开启 log_queries_not_using_indexes 后，slow log  文件不仅仅会记录慢查询日志，还会把查询过程中未使用索引或全表扫描的 SQL  记录到日志中，久而久之日志的空间便会变得越来越大，那么如何在线且安全的清空这些 slow log 日志，为磁盘释放空间呢？

MySQL 对于慢日志的输出方式支持两种，TABLE 和 FILE，查看方法如下：

```
mysql> show variables like '%log_output%'; 
+---------------+------------+
| Variable_name | Value      |
+---------------+------------+
| log_output    | FILE,TABLE |
+---------------+------------+
```

确认清楚输出方式后，可以分别对不同的输出方式选择不同的清空方法，本次将对两种清空方法共同介绍。

## 5.1. FILE 类型清空方法

**查询 slow query log 开启状态**

```
mysql> show variables like '%slow_query_log%';
+---------------------+-------------------------------------+
| Variable_name       | Value                               |
+---------------------+-------------------------------------+
| slow_query_log      | ON                                  |
| slow_query_log_file | /opt/mysql/data/3306/mysql-slow.log |
+---------------------+-------------------------------------+
```

**关闭 slow query log**

```
mysql> set global slow_query_log=0;
```

**确认关闭成功**

```
mysql> show variables like '%slow_query_log%';
+---------------------+-------------------------------------+
| Variable_name       | Value                               |
+---------------------+-------------------------------------+
| slow_query_log      | OFF                                 |
| slow_query_log_file | /opt/mysql/data/3306/mysql-slow.log |
+---------------------+-------------------------------------+
```

**对日志进行重命名或移除**

```
mv /opt/mysql/data/3306/mysql-slow.log /opt/mysql/data/3306/mysql-old-slow.log
```

**重新开启 slow query log**

```
mysql> set global slow_query_log=1;
```

**执行 SQL 进行验证**

```
mysql> select sleep(5);
+----------+
| sleep(5) |
+----------+
|        0 |
+----------+
```

**验证新生成文件记录成功**

```
cat /opt/mysql/data/3306/mysql-slow.log 
[root@DMP1 3306]# cat mysql-slow.log
/opt/mysql/base/5.7.31/bin/mysqld, Version: 5.7.31-log (MySQL Community Server (GPL)). started with:
Tcp port: 3306  Unix socket: /opt/mysql/data/3306/mysqld.sock
Time                 Id Command    Argument
# Time: 2021-01-05T13:26:44.001647+08:00
# User@Host: root[root] @ localhost []  Id: 81786
# Query_time: 5.000397  Lock_time: 0.000000 Rows_sent: 1  Rows_examined: 0
SET timestamp=1609824404;
select sleep(5);
```

**清理旧的 slowlog 文件**

```
mv /opt/mysql/data/3306/mysql-old-slow.log /mysqlback
```

## 5.2. TABLE 类型清空方法

**先关闭 slow query log**

```
mysql> set global slow_query_log=0;
```

**确认关闭成功**

```
mysql> show variables like 'slow_query_log';
+---------------------+-------------------------------------+
| Variable_name       | Value                               |
+---------------------+-------------------------------------+
| slow_query_log      | OFF                                 |
+---------------------+-------------------------------------+
```

**TABLE 类型的 slowlog 存放在 mysql.slow_log 表中,对 slow_log 进行重命名为 old_slow_log**

```
mysql> use mysql
mysql> ALTER TABLE slow_log RENAME old_slow_log;
```

**创建全新的 slow_log 文件拷贝原来 slow_log 文件的表结构**

```
mysql> CREATE TABLE slow_log LIKE old_slow_log;
```

**启动 slow query log**

```
mysql> SET GLOBAL slow_query_log = 1;
```

**测试验证**

```
mysql> select sleep(5);
+----------+
| sleep(5) |
+----------+
|        0 |
+----------+
mysql> select * from slow_log \G
*************************** 1. row ***************************
    start_time: 2021-01-05 13:49:13.864002
     user_host: root[root] @ localhost []
    query_time: 00:00:05.000322
     lock_time: 00:00:00.000000
     rows_sent: 1
 rows_examined: 0
            db: mysql
last_insert_id: 0
     insert_id: 0
     server_id: 874143039
      sql_text: select sleep(5)
     thread_id: 339487
```

**删除旧的 slow_log 表**

```
mysql> drop table old_slow_log;
```



# 参考资料

https://mp.weixin.qq.com/s/-2Xaw7UTvb6oFEGd3HApeg

https://mp.weixin.qq.com/s/CLN4e2W2PiSXW2PoUdDy1w

https://mp.weixin.qq.com/s/DFEAGKAfVWj-gzcwUJuwFQ

https://zhuanlan.zhihu.com/p/257975998