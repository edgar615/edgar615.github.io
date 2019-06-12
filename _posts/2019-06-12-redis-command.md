---
layout: post
title: redis命令工具
date: 2019-06-12
categories:
    - redis
comments: true
permalink: redis-command.html
---

# redis-server

```
Usage: 
		./redis-server [/path/to/redis.conf][options]
	   ./redis-server - (read config from stdin)
	   ./redis-server -v or --version
	   ./redis-server -h or --help
	   ./redis-server --test-memory <megabytes>

Examples:
       ./redis-server (run the server with default conf)
       ./redis-server /etc/redis/6379.conf
       ./redis-server --port 7777
       ./redis-server --port 7777 --slaveof 127.0.0.1 8888
       ./redis-server /etc/myredis.conf --loglevel verbose

Sentinel mode:
       ./redis-server /etc/sentinel.conf --sentinel
```

`redis-server --test-memory` 可以检测当前操作系统能否稳定地分配指定容量的内存给redis

	redis-server --test-memory 1024 检测能否提供1G内存
# redis-cli

- -r 重复执行命令
- -i 每隔几秒执行一次命令，单位是秒，如果需要按毫秒，使用小数如0.01表示10毫秒
- -x 从标准输入读取数据作为redis-cli的最后一个参数。 如：echo "world" | redis-cli -x set hello
- -c 连接集群节点是使用
- -a 密码
- --scan和-- pattern 扫描指定模式的键 相当于scan
- --slave 把当前客户端模拟成当前redis节点的从节点，可以用来获取当前redis节点的更新操作

```
[root@iZwz92yrze6r3mqbaygfk5Z redis-4.0.2]# src/redis-cli --slave
SYNC with master, discarding 194 bytes of bulk transfer...
SYNC done. Logging commands from master.
"PING"
"PING"
"SELECT","0"
"set","hello","w"
"PING"
"del","hello"
"PING"
```

- --rdb 生成并发送RDB持久化文件，保留在本地，可以使用它做定时备份
- --pipe 将命令封装成redis通信协议的数据格式，批量发送给redis执行
- --bigkeys 使用scan对redis的键进行采样，从中找到内存占用比较大的键值，这些键可能是系统的瓶颈
- --eval 执行lua脚本
- --latency 用于检测网络延时，它有三个选项
  - --latency 测试客户端到目标redis的网络延迟，示例redis-cli -h hostB --latency
  - --latency-history 以分时段的形式了解延迟信息，可以通过-i控制事件间隔
  - --latency-dis 以统计图表的形式输出延迟信息
- --stat 实时获取redis的统计信息

```
[root@iZwz92yrze6r3mqbaygfk5Z redis-4.0.2]# src/redis-cli --stat
------- data ------ --------------------- load -------------------- - child -
keys       mem      clients blocked requests            connections          
0          1.79M    1       0       36 (+0)             11          
0          1.79M    1       0       37 (+1)             11          
0          1.79M    1       0       38 (+1)             11          
0          1.79M    1       0       39 (+1)             11          
0          1.79M    1       0       40 (+1)             11 

```

- --raw和--no-raw --no-raw要求命令返回的结果必须是原始格式，--raw相反

示例：每隔1秒输出内存使用量

```
# src/redis-cli -r 5 -i 1 info | grep used_memory_human

used_memory_human:808.63K
used_memory_human:808.66K
used_memory_human:808.66K
used_memory_human:808.66K
used_memory_human:808.66K
```

# redis-benchmark
redis的基准测试

- -c 客户端的并发数量，默认50
- -n <requests> 客户端的请求总量 默认100000
- -q 仅显示requests per second信息
- -r 插入随机键，默认之后插入三个键 `counter:__rand__int__`,`mylist`,`key:__rand_int__`  -r会在key，counter键上加上一个12位的后缀，-r 10000表示只对后四位数做随机处理
- -P 每个请求pipeline的数据量，默认1
- -k <boolean> 客户端是否使用keepalive，1使用，0不使用，默认1
- -t 对指定的命令进行基准测试 如 -t get,set -q
- --csv 将结果按照csv格式输出

# 参考资料

《Redis开发与运维》



