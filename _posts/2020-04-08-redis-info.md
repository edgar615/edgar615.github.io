---
layout: post
title: redis info命令
date: 2020-04-08
categories:
    - redis
comments: true
permalink: redis-info.html
---

# info命令

INFO [section]

以一种易于解释（parse）且易于阅读的格式，返回关于 Redis 服务器的各种信息和统计数值。

通过给定可选的参数 section ，可以让命令只返回某一部分的信息：

## server
server 部分记录了 Redis 服务器的信息，它包含以下域：

	# Server
	redis_version:4.0.2 # redis版本
	redis_git_sha1:00000000 # Git SHA1
	redis_git_dirty:0 # Git dirty flag
	redis_build_id:19bba47a6c65947a
	redis_mode:standalone
	os:Linux 3.13.0-65-generic x86_64 # 服务器的宿主操作系统
	arch_bits:64 #架构（32 或 64 位）
	multiplexing_api:epoll #Redis所使用的事件处理机制
	atomicvar_api:atomic-builtin
	gcc_version:4.8.2 #编译 Redis 时所使用的 GCC 版本
	process_id:13818 #服务器进程的 PID
	run_id:53c7c487322fb0f857d9f8efbcfe7ca42352259b #Redis 服务器的随机标识符（用于 Sentinel 和集群）
	tcp_port:6379 #TCP/IP 监听端口
	uptime_in_seconds:11 #自 Redis 服务器启动以来，经过的秒数
	uptime_in_days:0 #自 Redis 服务器启动以来，经过的天数
	hz:10
	lru_clock:1210751 #以分钟为单位进行自增的时钟，用于 LRU 管理
	executable:/root/redis-4.0.2/src/redis-server
	config_file:/root/redis-4.0.2/redis.conf #配置文件

## Clients
clients 部分记录了已连接客户端的信息，它包含以下域：

    # Clients
    connected_clients:1 #已连接客户端的数量（不包括通过从属服务器连接的客户端）
    client_longest_output_list:0 #当前连接的客户端当中，最长的输出列表
    client_biggest_input_buf:0 #当前连接的客户端当中，最大输入缓存
    blocked_clients:0 正在等待阻塞命令（BLPOP、BRPOP、BRPOPLPUSH）的客户端的数量

## Memory
memory 部分记录了服务器的内存信息，它包含以下域：

    # Memory
    used_memory:828976 #由 Redis 分配器分配的内存总量，以字节（byte）为单位
    used_memory_human:809.55K #以人类可读的格式返回 Redis 分配的内存总量
    used_memory_rss:8208384 #从操作系统的角度，返回 Redis 已分配的内存总量（俗称常驻集大小）。这个值和 top 、 ps 等命令的输出一致。
    used_memory_rss_human:7.83M
    used_memory_peak:828976 #Redis 的内存消耗峰值（以字节为单位）
    used_memory_peak_human:809.55K
    used_memory_peak_perc:100.13%
    used_memory_overhead:815742
    used_memory_startup:766112
    used_memory_dataset:13234
    used_memory_dataset_perc:21.05%
    total_system_memory:4145299456
    total_system_memory_human:3.86G
    used_memory_lua:37888 #Lua 引擎所使用的内存大小（以字节为单位）
    used_memory_lua_human:37.00K
    maxmemory:0
    maxmemory_human:0B
    maxmemory_policy:noeviction
    mem_fragmentation_ratio:9.90 #used_memory_rss 和 used_memory 之间的比率
    mem_allocator:jemalloc-4.0.3 #在编译时指定的， Redis 所使用的内存分配器。可以是 libc 、 jemalloc 或者 tcmalloc
    active_defrag_running:0
    lazyfree_pending_objects:0

在理想情况下， used_memory_rss 的值应该只比 used_memory 稍微高一点儿。
当 rss > used ，且两者的值相差较大时，表示存在（内部或外部的）内存碎片。
内存碎片的比率可以通过 mem_fragmentation_ratio 的值看出。
当 used > rss 时，表示 Redis 的部分内存被操作系统换出到交换空间了，在这种情况下，操作可能会产生明显的延迟。

	Because Redis does not have control over how its allocations are mapped to memory pages, high used_memory_rss is often the result of a spike in memory usage.

当 Redis 释放内存时，分配器可能会，也可能不会，将内存返还给操作系统。
如果 Redis 释放了内存，却没有将内存返还给操作系统，那么 used_memory 的值可能和操作系统显示的 Redis 内存占用并不一致。
查看 used_memory_peak 的值可以验证这种情况是否发生。

## Persistence
persistence 部分记录了跟 RDB 持久化和 AOF 持久化有关的信息，它包含以下域：

# Persistence
    loading:0 #一个标志值，记录了服务器是否正在载入持久化文件。
    rdb_changes_since_last_save:0 #距离最近一次成功创建持久化文件之后，经过了多少秒。
    rdb_bgsave_in_progress:0 #一个标志值，记录了服务器是否正在创建 RDB 文件。
    rdb_last_save_time:1511160180 #最近一次成功创建 RDB 文件的 UNIX 时间戳。
    rdb_last_bgsave_status:ok #一个标志值，记录了最近一次创建 RDB 文件的结果是成功还是失败。
    rdb_last_bgsave_time_sec:-1 #记录了最近一次创建 RDB 文件耗费的秒数。
    rdb_current_bgsave_time_sec:-1 #如果服务器正在创建 RDB 文件，那么这个域记录的就是当前的创建操作已经耗费的秒数。
    rdb_last_cow_size:0
    aof_enabled:0 #一个标志值，记录了 AOF 是否处于打开状态。
    aof_rewrite_in_progress:0 #个标志值，记录了服务器是否正在创建 AOF 文件。
    aof_rewrite_scheduled:0 #一个标志值，记录了在 RDB 文件创建完毕之后，是否需要执行预约的 AOF 重写操作。
    aof_last_rewrite_time_sec:-1 #最近一次创建 AOF 文件耗费的时长。
    aof_current_rewrite_time_sec:-1 #如果服务器正在创建 AOF 文件，那么这个域记录的就是当前的创建操作已经耗费的秒数。
    aof_last_bgrewrite_status:ok #一个标志值，记录了最近一次创建 AOF 文件的结果是成功还是失败。
    aof_last_write_status:ok #一个标志值，记录了最近一次创建 AOF 文件的结果是成功还是失败。
    aof_last_cow_size:0

如果 AOF 持久化功能处于开启状态，那么这个部分还会加上以下域：
​    
    aof_current_size : AOF 文件目前的大小。
    aof_base_size : 服务器启动时或者 AOF 重写最近一次执行之后，AOF 文件的大小。
    aof_pending_rewrite : 一个标志值，记录了是否有 AOF 重写操作在等待 RDB 文件创建完毕之后执行。
    aof_buffer_length : AOF 缓冲区的大小。
    aof_rewrite_buffer_length : AOF 重写缓冲区的大小。
    aof_pending_bio_fsync : 后台 I/O 队列里面，等待执行的 fsync 调用数量。
    aof_delayed_fsync : 被延迟的 fsync 调用数量。

## Stats
stats 部分记录了一般统计信息，它包含以下域：
​    
	# Stats
	total_connections_received:1 #服务器已接受的连接请求数量。
	total_commands_processed:1 #服务器已执行的命令数量。
	instantaneous_ops_per_sec:0 #服务器每秒钟执行的命令数量。
	total_net_input_bytes:31
	total_net_output_bytes:10162
	instantaneous_input_kbps:0.00
	instantaneous_output_kbps:0.00
	rejected_connections:0 #因为最大客户端数量限制而被拒绝的连接请求数量。
	sync_full:0
	sync_partial_ok:0
	sync_partial_err:0
	expired_keys:0 # 因为过期而被自动删除的数据库键数量。
	evicted_keys:0 #因为最大内存容量限制而被驱逐（evict）的键数量。
	keyspace_hits:0 #查找数据库键成功的次数。
	keyspace_misses:0 #查找数据库键失败的次数。
	pubsub_channels:0 #目前被订阅的频道数量
	pubsub_patterns:0 #目前被订阅的模式数量
	latest_fork_usec:0 #最近一次 fork() 操作耗费的毫秒数
	migrate_cached_sockets:0
	slave_expires_tracked_keys:0
	active_defrag_hits:0
	active_defrag_misses:0
	active_defrag_key_hits:0
	active_defrag_key_misses:0

## Replication
replication : 主/从复制信息

    # Replication
    role:master #如果当前服务器没有在复制任何其他服务器，那么这个域的值就是 master ；否则的话，这个域的值就是 slave 。注意，在创建复制链的时候，一个从服务器也可能是另一个服务器的主服务器。
    connected_slaves:1 #已连接的从服务器数量。
    slave0:ip=127.0.0.1,port=6380,state=online,offset=690,lag=0 #从服务器的IP、断开，状态、复制偏移量，lag是与从节点最后一次通信延迟的秒数，正常延迟应该在0和1之间
    master_replid:d7ffc06a335aa6d77dad778b8b3088d6e1212fbd #运行ID，40位的16进制字符串，用来识别redis节点
    master_replid2:0000000000000000000000000000000000000000
    master_repl_offset:690 #主节点的命令偏移量
    second_repl_offset:-1
    repl_backlog_active:1 # 开启复制缓冲区
    repl_backlog_size:1048576 #缓冲区最大长度
    repl_backlog_first_byte_offset:1 #起始偏移量，计算当前缓冲区的可用范围
    repl_backlog_histlen:690 #已保存数据的有效长度

如果当前服务器是一个从服务器的话，那么这个部分还会加上以下域：

    master_host:127.0.0.1 #主服务器的 IP 地址
    master_port:6379 #主服务器的 TCP 监听端口号
    master_link_status:up #复制连接当前的状态， up 表示连接正常， down 表示连接断开
    master_last_io_seconds_ago:3 #距离最近一次与主服务器进行通信已经过去了多少秒钟
    master_sync_in_progress:0 #一个标志值，记录了主服务器是否正在与这个从服务器进行同步
    slave_repl_offset:2902 #自身的偏移量
    slave_priority:100
    slave_read_only:1
    connected_slaves:0
    master_replid:d7ffc06a335aa6d77dad778b8b3088d6e1212fbd
    master_replid2:0000000000000000000000000000000000000000
    master_repl_offset:2902
    second_repl_offset:-1
    repl_backlog_active:1 
    repl_backlog_size:1048576 
    repl_backlog_first_byte_offset:1
    repl_backlog_histlen:2902

如果同步操作正在进行，那么这个部分还会加上以下域：

    master_sync_left_bytes : 距离同步完成还缺少多少字节数据。
    master_sync_last_io_seconds_ago : 距离最近一次因为 SYNC 操作而进行 I/O 已经过去了多少秒。

如果主从服务器之间的连接处于断线状态，那么这个部分还会加上以下域：

	master_link_down_since_seconds : 主从服务器连接断开了多少秒。

对于每个从服务器，都会添加以下一行信息：

	slaveXXX : ID、IP 地址、端口号、连接状态

## CPU
cpu 部分记录了 CPU 的计算量统计信息，它包含以下域：
​    
    ## CPU
    used_cpu_sys:0.01 #Redis 服务器耗费的系统 CPU 
    used_cpu_user:0.00 #Redis 服务器耗费的用户 CPU 。
    used_cpu_sys_children:0.00 #后台进程耗费的系统 CPU 。
    used_cpu_user_children:0.00 #后台进程耗费的系统 CPU 。

## commandstats
commandstats 部分记录了各种不同类型的命令的执行统计信息，比如命令执行的次数、命令耗费的 CPU 时间、执行每个命令耗费的平均 CPU 时间等等。对于每种类型的命令，这个部分都会添加一行以下格式的信息：
​    
	cmdstat_XXX:calls=XXX,usec=XXX,usecpercall=XXX

## Cluster
cluster 部分记录了和集群有关的信息，它包含以下域：

	# Cluster
	cluster_enabled:0 #一个标志值，记录集群功能是否已经开启。

## keyspace
    keyspace 部分记录了数据库相关的统计信息，比如数据库的键数量、数据库已经被删除的过期键数量等。对于每个数据库，这个部分都会添加一行以下格式的信息：

	dbXXX:keys=XXX,expires=XXX

除上面给出的这些值以外， section 参数的值还可以是下面这两个：

    all : 返回所有信息
    default : 返回默认选择的信息

当不带参数直接调用 INFO 命令时，使用 default 作为默认参数。