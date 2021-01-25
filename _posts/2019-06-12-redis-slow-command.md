---
layout: post
title: redis慢查询
date: 2019-06-12
categories:
    - redis
comments: true
permalink: redis-slow-command.html
---

Redis客户端执行一条命令分为4个部分：

1. 发送命令
2. 命令排队
3. 命令执行
4. 返回结果

慢查询只统计步骤3的时间，所以没有慢查询并不代表客户端没有超时问题

# 配置项

```
	# The following time is expressed in microseconds, so 1000000 is equivalent to one second. Note that a negative number disables the slow log, while a value of zero forces the logging of every command.
	# 阈值，单位微秒，默认值1秒，为0时会记录所有命令，为-1时不会记录所有命令
	slowlog-log-slower-than 10000 
	
	# 最多存储多少条慢查询日志，默认值128，redis使用一个list类型存储日志
	# There is no limit to this length. Just be aware that it will consume memory. You can reclaim memory used by the slow log with SLOWLOG RESET.
	slowlog-max-len 128 
```

# 测试
**设置慢查询**

```
127.0.0.1:6379> config set slowlog-log-slower-than 0
OK
127.0.0.1:6379> config set slowlog-max-len 5
OK
127.0.0.1:6379> CONFIG REWRITE
```

**查询慢查询**  `slowlog get [n]` 参数n指定条数

```
127.0.0.1:6379> slowlog get 3
1) 1) (integer) 3 //标识ID
   2) (integer) 1510128238 //发生时间戳
   3) (integer) 38 //命令耗时
   4) 1) "slowlog" //执行命令
	  2) "get"
	  3) "2"
   5) "127.0.0.1:42638" //客户端
   6) ""
2) 1) (integer) 2
   2) (integer) 1510128181
   3) (integer) 3
   4) 1) "CONFIG"
	  2) "REWRITE"
   5) "127.0.0.1:42638"
   6) ""
3) 1) (integer) 1
   2) (integer) 1510128172
   3) (integer) 8
   4) 1) "config"
	  2) "set"
	  3) "slowlog-max-len"
	  4) "5"
   5) "127.0.0.1:42638"
   6) ""
```

**慢查询列表长度** `slowlog len`

```
127.0.0.1:6379> slowlog len
(integer) 5
```

**慢查询重置** `slowlog reset`

```
127.0.0.1:6379> slowlog reset
OK
127.0.0.1:6379> slowlog len
(integer) 1
```

# 配置建议

- slowlog-log-slower-than 建议设10毫秒，并根据并发量调整（最好根据自身redis的基线测试情况设置）
- slowlog-max-len 1000 线上可以调大列表长度，因为redis会自动截断过长的长度，不会占用过多内存
	

慢查询只记录命令执行时间，并不包括命令排队和网络传输时间，因此客户端执行命令的时间会大于命令实际执行时间。
因为命令执行排队机制，慢查询会导致其他命令级联阻塞。因此当客户端出现请求超时时，需要检查该时间点是否有对应的慢查询。

由于慢查询日志是一个先进先出队列，在慢查询比较多的情况下，可能会丢失布防慢查询命令，为了防止这种情况发生，可以定期执行slow get命令将慢查询日志持久化到其他存储中

# 参考资料

《Redis开发与运维》

