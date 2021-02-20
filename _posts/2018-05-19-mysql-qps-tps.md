---
layout: post
title: MySQL运维与监控（4）- 查看tps和qps
date: 2018-05-18
categories:
    - MySQL
comments: true
permalink: mysql-tps.html
---
# QPS

Queries Per Second：每秒查询数，一台数据库每秒能够处理的查询次数

方式1，例如采样10秒内的查询次数，那么先查询一次Queries值（Q1），等待10秒，再查询一次Queries值（Q2） QPS = (Q2 - Q1) / 10 

```
mysql> show status like '%queries%';
+-------------------------+----------+
| Variable_name           | Value    |
+-------------------------+----------+
| Qcache_queries_in_cache | 0        |
| Queries                 | 64961461 |
| Slow_queries            | 0        |
+-------------------------+----------+
3 rows in set (0.00 sec)
```

方式2，QPS=Queries / Uptime 

```
mysql> show status like '%queries%';
+-------------------------+----------+
| Variable_name           | Value    |
+-------------------------+----------+
| Qcache_queries_in_cache | 0        |
| Queries                 | 64961461 |
| Slow_queries            | 0        |
+-------------------------+----------+
3 rows in set (0.00 sec)

mysql> show status like '%uptime%'; 
+---------------------------+---------+
| Variable_name             | Value   |
+---------------------------+---------+
| Uptime                    | 6713324 |
| Uptime_since_flush_status | 6713324 |
+---------------------------+---------+
2 rows in set (0.00 sec)
```

# TPS

Transactions Per Second：每秒处理事务数

```
TPS = (Com_commit + Com_rollback) / Seconds
TPS=(Com_insert + Com_delete + Com_update) / Seconds
```

还有两个比较重要的参数

- Threads_connected 当前连接的线程的个数
-  Threads_running 运行状态的线程的个数 



通过脚本计算QPS

```
#!/bin/bash
mysqladmin -uroot -p'123456' extended-status -i1|awk \
'BEGIN{flag=0;
print "";
print "QPS   TPS    Threads_con Threads_run ";
print "------------------------------------- "}
$2 ~ /Queries$/            {q=$4-lq;lq=$4;}
$2 ~ /Com_commit$/         {c=$4-lc;lc=$4;}
$2 ~ /Com_rollback$/       {r=$4-lr;lr=$4;}
$2 ~ /Threads_connected$/  {tc=$4;}
$2 ~ /Threads_running$/    {tr=$4;

if(flag==0){
    flag=1; count=0
}else {
    printf "%-6d %-8d %-10d %d \n", q,c+r,tc,tr;
}

}'
```

awk是代码中的重点，mysqladmin 的执行结果通过管道传给 awk 进行分析

```
'BEGIN{flag=0;
print "";
print "QPS   TPS    Threads_con Threads_run ";
print "------------------------------------- "}
```

这部分是初始设置，打印出表头
flag=0 是设置一个标识位，后面用到

```
$2 ~ /Queries$/  {q=$4-lq;lq=$4;}
其中 $2 $4代表某列的内容
```

awk是按行分析并按空格分割的，例如行信息为：
| Queries | 213263713 |
按空格分割后得到5列：
‘|’, ‘Queries’, ‘|’, ‘213263713’, ‘|’

```
$2 : Queries
$4 : 213263713
```

那么这句的意思就是：
当第2列的值匹配‘Queries’时，
变量q = 第4列的值 - 变量lq的值，
变量lq = 第4列的值
变量q 就是 QPS值，用这一次的 Queries值 减去 上一次的值

```
$2 ~ /Com_commit$/   {c=$4-lc;lc=$4;}
$2 ~ /Com_rollback$/   {r=$4-lr;lr=$4;}
$2 ~ /Threads_connected$/  {tc=$4;}
$2 ~ /Threads_running$/    {tr=$4;
```

这几句的意思与上一句类似

```
if(flag==0){
    flag=1; 
}
```

这里用到了flag这个标识位，意思是对第一次的分析结果什么都不做，因为这句
{q=4−lq;lq=

4;}
q=$4-lq; 中的 lq 在第一次分析中还没有值

```
else {
    printf "%-6d %-8d %-10d %d \n", q,c+r,tc,tr;
}
```

这部分就是打印统计结果信息