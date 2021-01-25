---
layout: post
title: redis延迟分析
date: 2020-04-08
categories:
    - redis
comments: true
permalink: redis-latency.html
---

延迟（latency）指从客户端发出一条命令到客户端接受到该命令的反馈所用的最长响应时间

# 1. 计算延迟时间
如果你正在经历响应延迟问题，你或许能够根据应用程序的具体情况算出它的延迟响应时间，或者你的延迟问题非常明显，宏观看来，一目了然。不管怎样吧，用redis-cli可以算出一台Redis 服务器的到底延迟了多少毫秒。

```
redis-cli --latency -h `host` -p `port`
```

示例

```
# src/redis-cli --latency -i 3
min: 0, max: 4, avg: 0.29 (3903 samples)
```

```
# src/redis-cli --intrinsic-latency 120
Max latency so far: 1 microseconds.
Max latency so far: 9 microseconds.
Max latency so far: 20 microseconds.
Max latency so far: 67 microseconds.
Max latency so far: 73 microseconds.
Max latency so far: 90 microseconds.
Max latency so far: 97 microseconds.
Max latency so far: 106 microseconds.
Max latency so far: 197 microseconds.
Max latency so far: 386 microseconds.
Max latency so far: 458 microseconds.
Max latency so far: 1224 microseconds.
Max latency so far: 5067 microseconds.
Max latency so far: 8932 microseconds.
Max latency so far: 15707 microseconds.

3553236533 total runs (avg latency: 0.0338 microseconds / 33.77 nanoseconds per run).
Worst run took 465089x longer than the average latency.
```

# 2. 延迟配置

开启延迟监控

```
CONFIG SET latency-monitor-threshold 100 单位为毫秒
```

默认情况latency-monitor-threshold为0, 即延迟监控是关闭的。延迟监控功能占用内存很小, 不过对于性能良好的redis也没有必要开启。

相关命令

- **LATENCY LATEST** 返回所有事件的最新延迟样本
- **LATENCY HISTORY event** 返回最多160条的给定 event 的延迟时间序列（延迟发生时的时间戳和延迟毫秒数）
- **LATENCY RESET event** 重置一个或多个 events 的延迟时间序列数据为零，如果不指定参数 event，则表示重置所有的 events。
- **LATENCY GRAPH event** 以文本图表方式展示
- **LATENCY DOCTOR** 回复人类可读的延迟分析报告
- **LATENCY HELP** 查看使用帮助

支持的事件

- command	常规命令
- fast-command	时间复杂度为“O(1)”和“O(log N)”的快命令
- fork	系统调用 fork
- aof-stat	系统调用 stat
- aof-write	系统调用 write
- aof-rename	系统调用 rename
- aof-fsync-always	设置“appendfsync allways”时的系统调用 fsync
- aof-write-active-child	子进程执行的系统调用 fsync
- rdb-unlink-temp-file	系统调用 unlink
- active-defrag-cycle	主动碎片整理周期
- aof-rewrite-diff-write	
- aof-write-alone	主进程执行的 fsync 系统调用
- aof-write-pending-fsync	
- expire-cycle	过期周期
- eviction-cycle	淘汰周期

# 3. 延迟分析

从业务服务到 Redis 这条链路变慢的原因可能也有 2 个：

- 业务服务器到 Redis 服务器之间的网络存在问题，例如网络线路质量不佳，网络数据包在传输时存在延迟、丢包等情况
- Redis 本身存在问题，需要进一步排查是什么原因导致 Redis 变慢。

对于Redis本身的问题，我们需要首先了解 Redis 在生产环境服务器上的基线性能。因为 Redis 在不同的软硬件环境下，它的性能是各不相同的。

例如，我的机器配置比较低，当延迟为 2ms 时，我就认为 Redis 变慢了，但是如果你的硬件配置比较高，那么在你的运行环境下，可能延迟是 0.5ms 时就可以认为 Redis 变慢了。

所以，你只有了解了你的 Redis 在生产环境服务器上的基线，才能进一步评估，当其延迟达到什么程度时，才认为 Redis 确实变慢了。

我们可以把它和 Redis 运行时的延迟结合起来，再进一步判断 Redis 性能是否变慢了。一般来说，你要把运行时延迟和基线性能进行对比，**如果观察到的 Redis 运行时延迟是其基线性能的 2 倍及以上，就可以认定 Redis 变慢了。**

## 3.1. 网络通信延迟

当用户连接到Redis通过TCP/IP连接或Unix域连接，千兆网络的典型延迟大概200us，而Unix域socket可能低到30us。这完全基于你的网络和系统硬件。在通信本身之上，系统增加了更多的延迟（线程调度，CPU缓存，NUMA替换等等）。系统引起的延迟在虚拟机环境远远高于在物理机器环境。 

是即使Redis处理大多数命令在微秒之下，客户机和服务器之间的交互也必然消耗系统相关的延迟。 

**建议**

- 尽可能的使用物理机而不是虚拟机来做服务器
- 不要经常的connect/disconnect与服务器的连接（尤其是对基于web的应用），尽可能的延长与服务器连接的时间。（长连接）
- 如果你的客户端和服务器在同一台主机上，则使用Unix域套接字
- 优先使用聚合命令(MSET/MGET), 而不是pipeline
- 优先使用pipeline, 而不是频繁发送命令(多次网络往返)
- 对不适合使用pipeline的命令, 可以考虑使用lua脚本

## 3.2. 使用复杂度过高的命令

Redis 使用了单线程的处理所有的客户端请求。**当一个请求执行得很慢，其他的客户端调用就必须等待这个请求执行完毕**。GET or SET or LPUSH 等命令执行时间是常数, 不过类似 SORT, LREM, SUNION等操作多个元素的命令执行时间是O(N)

> 如何查找慢查询？slowlog-log-slower-than，slowlog-max-len，slowlog get
>
> https://edgar615.github.io/redis-slow-command.html

通过查看慢日志，我们就可以知道在什么时间点，执行了哪些命令比较耗时。如果应用程序执行的 Redis 命令有以下特点，那么有可能会导致操作延迟变大：

- 经常使用 O(N) 以上复杂度的命令，例如 SORT、SUNION、ZUNIONSTORE 聚合类命令
- 使用 O(N) 复杂度的命令，但 N 的值非常大

第一种情况导致变慢的原因在于，Redis 在操作内存数据时，时间复杂度过高，要花费更多的 CPU 资源。

第二种情况导致变慢的原因在于，Redis 一次需要返回给客户端的数据过多，更多时间花费在数据协议的组装和网络传输过程中。

我们还可以从资源使用率层面来分析，如果应用程序操作 Redis 的 OPS 不是很大，但 Redis 实例的 **CPU 使用率却很高**，那么很有可能是使用了复杂度过高的命令导致的。

**建议**

- 减少多元素慢命令的使用
- 对于数据的聚合操作，放在客户端做
- 执行 O(N) 命令，保证 N 尽量的小（推荐 N <= 300），每次获取尽量少的数据，让 Redis 可以及时处理返回
-  对于 KEYS命令只能用于线下调试, 生产环境可以使用 SCAN, SSCAN, HSCAN and ZSCAN等命令代替
- 使用主从复制, 将慢的命令放到复制机器上执行
- 使用SLOWLOG 分析慢查询

##  3.3. 操作bigkey

如果查询慢日志发现，并不是复杂度过高的命令导致的，而都是 SET / DEL 这种简单命令出现在慢日志中，那么就要怀疑你的实例否写入了 bigkey。

Redis 在写入数据时，需要为新的数据分配内存，相对应的，当从 Redis 中删除数据时，它会释放对应的内存空间。

如果一个 key 写入的 value 非常大，那么 Redis 在**分配内存时就会比较耗时**。同样的，当删除这个 key 时，**释放内存也会比较耗时**，这种类型的 key 我们一般称之为 bigkey。

此时，我们需要检查业务代码，是否存在写入 bigkey 的情况。需要评估写入一个 key 的数据大小，尽量避免一个 key 存入过大的数据。

**建议：**

- 业务应用尽量避免写入 bigkey
- 如果 Redis 是 4.0 以上版本，用 UNLINK 命令替代 DEL，此命令可以把释放 key 内存的操作，放到后台线程中去执行，从而降低对 Redis 的影响
- 如果 Redis 是 6.0 以上版本，可以开启 lazy-free 机制（lazyfree-lazy-user-del = yes），在执行 DEL 命令时，释放内存也会放到后台线程中执行

## 3.4.  集中过期

如果平时在操作 Redis 时，并没有延迟很大的情况发生，但在某个时间点突然出现一波延时，其现象表现为：**变慢的时间点很有规律，例如某个整点，或者每间隔多久就会发生一波延迟。**如果出现这种情况，那么我们就需要排查一下，业务代码中是否存在设置大量 key 集中过期的情况。

如果有大量的 key 在某个固定时间点集中过期，在这个时间点访问 Redis 时，就有可能导致延时变大。

redis有两种方式来去除过期的key：

- **被动过期：**只有当访问某个 key 时，才判断这个 key 是否已过期，如果已过期，则从实例中删除
- **主动过期：**Redis 内部维护了一个定时任务，默认每隔 100 毫秒（1秒10次）就会从全局的过期哈希表中随机取出 20 个  key，然后删除其中过期的 key，如果**过期 key 的比例超过了 25%**，则继续重复此过程，直到过期 key 的比例下降到 25%  以下，或者这次任务的执行耗时**超过了 25 毫秒**，才会退出循环

**这个主动过期 key 的定时任务，是在 Redis 主线程中执行的**。而且主动过期的算法是自适应的，如果发现有超过指定数量25%的key已经过期就会循环执行。通常来讲如果有很多key在同一秒过期，超过了所有key的25%，redis就会阻塞直到过期key的比例下降到25%以下。此时就会出现，**应用访问 Redis 延时变大。**

如果此时需要过期删除的是一个 bigkey，那么这个耗时会更久。而且，**这个操作延迟的命令并不会记录在慢日志中**。

因为慢日志中**只记录一个命令真正操作内存数据的耗时**，而 Redis 主动删除过期 key 的逻辑，是在命令真正执行之前执行的。

所以如果慢日志中没有操作耗时的命令，但应用程序却感知到了延迟变大，其实时间都花费在了删除过期 key 上，这种情况我们需要尤为注意。

**建议：**

- 集中过期 key 增加一个随机过期时间，把集中过期的时间打散，降低 Redis 清理过期 key 的压力

- 如果Redis 是 4.0 以上版本，可以开启 lazy-free 机制，当删除过期 key 时，把释放内存的操作放到后台线程中执行，避免阻塞主线程
- 通过info命令监控expired_keys ，它代表整个实例到目前为止，累计删除过期 key 的数量。**当这个指标在很短时间内出现了突增**，需要及时报警出来，然后与业务应用报慢的时间点进行对比分析，确认时间是否一致，如果一致，则可以确认确实是因为集中过期 key 导致的延迟变大。

## 3.5. 内存达到上限

当我们把 Redis 当做纯缓存使用时，通常会给这个实例设置一个内存上限 **maxmemory**，然后设置一个数据淘汰策略。而当实例的内存达到了 maxmemory 后，我们可能会发现，在此之后每次写入新数据，操作延迟变大了。

原因在于，当 Redis 内存达到 maxmemory 后，每次写入新的数据之前，**Redis 必须先从实例中踢出一部分数据，让整个实例的内存维持在 maxmemory 之下**，然后才能把新数据写进来。

这个踢出旧数据的逻辑也是需要消耗时间的，而具体耗时的长短，要取决于你配置的淘汰策略：

- allkeys-lru：不管 key 是否设置了过期，淘汰最近最少访问的 key
- volatile-lru：只淘汰最近最少访问、并设置了过期时间的 key
- allkeys-random：不管 key 是否设置了过期，随机淘汰 key
- volatile-random：只随机淘汰设置了过期时间的 key
- allkeys-ttl：不管 key 是否设置了过期，淘汰即将过期的 key
- noeviction：不淘汰任何 key，实例内存达到 maxmeory 后，再写入新数据直接返回错误
- allkeys-lfu：不管 key 是否设置了过期，淘汰访问频率最低的 key（4.0+版本支持）
- volatile-lfu：只淘汰访问频率最低、并设置了过期时间 key（4.0+版本支持）

具体使用哪种策略，我们需要根据具体的业务场景来配置。

一般最常使用的是 allkeys-lru / volatile-lru 淘汰策略，它们的处理逻辑是，每次从实例中随机取出一批  key（这个数量可配置），然后淘汰一个最少访问的 key，之后把剩下的 key 暂存到一个池子中，继续随机取一批 key，并与之前池子中的  key 比较，再淘汰一个最少访问的 key。以此往复，直到实例内存降到 maxmemory 之下。

需要注意的是，Redis 的淘汰数据的逻辑与删除过期 key 的一样，**也是在命令真正执行之前执行的**，也就是说它也会增加操作 Redis 的延迟，而且，写QPS 越高，延迟也会越明显。

另外，如果此时你的 Redis 实例中还存储了 bigkey，那么**在淘汰删除 bigkey 释放内存时，也会耗时比较久**。

**建议：**

- 避免存储 bigkey，降低释放内存的耗时
- 淘汰策略改为随机淘汰，随机淘汰比 LRU 要快很多（视业务情况调整）
- 拆分实例，把淘汰 key 的压力分摊到多个实例上
- 如果使用的是 Redis 4.0 以上版本，开启 layz-free 机制，把淘汰 key 释放内存的操作放到后台线程中执行（配置 lazyfree-lazy-eviction = yes）

## 3.6. fork耗时严重

如果你发现，**操作 Redis 延迟变大，都发生在 Redis 后台 RDB 和 AOF rewrite 期间**，就你需要排查，在这期间有可能导致变慢的情况。

生成RDB或者AOF会使redis 主线程fork后台线程, 这会造成一定延迟。可以在 Redis 上执行 INFO 命令，查看 latest_fork_usec 项，单位微秒。

```
# 上一次 fork 耗时，单位微秒
latest_fork_usec:59477
```

这个时间就是主进程在 fork 子进程期间，整个实例阻塞无法处理客户端请求的时间。如果发现这个耗时很久，就要警惕起来了。

对于运行在一个linux/AMD64系统上的实例来说，内存会按照每页4KB的大小分页。为了实现虚拟地址到物理地址的转换，每一个进程将会存储一个分页表（树状形式表现），分页表将至少包含一个指向该进程地址空间的指针。所以一个空间大小为24GB的redis实例，需要的分页表大小为  24GB/4KB*8 = 48MB。

当一个后台的save命令执行时，实例会启动新的线程去申请和拷贝48MB的内存空间。这将消耗一些时间和CPU资源，尤其是在虚拟机上申请和初始化大块内存空间时，消耗更加明显。

而且这个 fork 过程会消耗大量的 CPU 资源，在完成 fork 之前，整个 Redis 实例会被阻塞住，无法处理任何客户端请求。如果此时你的 CPU 资源本来就很紧张，那么 fork 的耗时会更长，甚至达到秒级，这会严重影响 Redis 的性能。

不同系统中的Fork时间

    Linux beefy VM on VMware 6.0GB RSS forked 77 微秒 (每GB 12.8 微秒 ).
    Linux running on physical machine (Unknown HW) 6.1GB RSS forked 80 微秒(每GB 13.1微秒)
    Linux running on physical machine (Xeon @ 2.27Ghz) 6.9GB RSS forked into 62 微秒 (每GB 9 微秒).
    Linux VM on 6sync (KVM) 360 MB RSS forked in 8.2 微秒 (每GB 23.3 微秒).
    Linux VM on EC2 (Xen) 6.1GB RSS forked in 1460 微秒 (每GB 239.3 微秒).
    Linux VM on Linode (Xen) 0.9GBRSS forked into 382 微秒 (每GB 424 微秒).

**建议：**

- 控制 Redis 实例的内存：尽量在 10G 以下，执行 fork 的耗时与实例大小有关，实例越大，耗时越久
- 合理配置数据持久化策略：在 slave 节点执行 RDB 备份，推荐在低峰期执行，而对于丢失数据不敏感的业务（例如把 Redis 当做纯缓存使用），可以关闭 AOF 和 AOF rewrite
- Redis 实例不要部署在虚拟机上：fork 的耗时也与系统也有关，虚拟机比物理机耗时更久
- 降低主从库全量同步的概率：适当调大 repl-backlog-size 参数，避免主从全量同步

## 3.7. 开启了内存大页

除了上面讲到的子进程 RDB 和 AOF rewrite 期间，fork 耗时导致的延时变大之外，这里还有一个方面也会导致性能问题，这就是操作系统是否开启了**内存大页机制**。

应用程序向操作系统申请内存时，是按**内存页**进行申请的，而常规的内存页大小是 4KB。Linux 内核从 2.6.38 开始，支持了**内存大页机制**，该机制允许应用程序以 2MB 大小为单位，向操作系统申请内存。**应用程序每次向操作系统申请的内存单位变大了，但这也意味着申请内存的耗时变长。**

当 Redis 在执行后台 RDB 和 AOF rewrite 时，采用 fork 子进程的方式来处理。但主进程 fork 子进程后，此时的**主进程依旧是可以接收写请求的**，而进来的写请求，会采用 Copy On Write（写时复制）的方式操作内存数据。

也就是说，主进程一旦有数据需要修改，Redis 并不会直接修改现有内存中的数据，而是**先将这块内存数据拷贝出来，再修改这块新内存的数据**，这就是所谓的「写时复制」。

这样做的好处是，父进程有任何写操作，并不会影响子进程的数据持久化，子进程只持久化 fork 这一瞬间整个实例中的所有数据即可，不关心新的数据变更，因为子进程只需要一份内存快照，然后持久化到磁盘上。

但是主进程在拷贝内存数据时，这个阶段就涉及到新内存的申请，如果此时操作系统开启了内存大页，那么在此期间，客户端即便只修改 10B 的数据，**Redis 在申请内存时也会以 2MB 为单位向操作系统申请，申请内存的耗时变长，进而导致每个写请求的延迟增加，影响到 Redis 性能。**

同样地，如果这个写请求操作的是一个 bigkey，那主进程在拷贝这个 bigkey 内存块时，一次申请的内存会更大，时间也会更久。

```
# cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never
# 如果输出选项是 always，就表示目前开启了内存大页机制
# 关闭大页
# echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

## 3.8. AOF设置不合理

当 Redis 开启 AOF 后，其工作原理如下：

- Redis 执行写命令后，把这个命令写入到 AOF 文件内存中（write 系统调用）
- Redis 根据配置的 AOF 刷盘策略，把 AOF 内存数据刷到磁盘上（fsync 系统调用）

为了保证 AOF 文件数据的安全性，Redis 提供了 3 种刷盘机制：

- appendfsync always：主线程每次执行写操作后立即刷盘，此方案会占用比较大的磁盘 IO 资源，但数据安全性最高
- appendfsync no：主线程每次写操作只写内存就返回，内存数据什么时候刷到磁盘，交由操作系统决定，此方案对性能影响最小，但数据安全性也最低，Redis 宕机时丢失的数据取决于操作系统刷盘时机
- appendfsync everysec：主线程每次写操作只写内存就返回，然后由后台线程每隔 1 秒执行一次刷盘操作（触发fsync系统调用），此方案对性能影响相对较小，但当 Redis 宕机时会丢失 1 秒的数据

如果你的 AOF 配置为 appendfsync always，那么 Redis 每处理一次写操作，都会把这个命令写入到磁盘中才返回，整个过程都是在主线程执行的，这个过程必然会加重 Redis 写负担。因为操作磁盘要比操作内存慢几百倍，采用这个配置会严重拖慢 Redis 的性能。

即使我们使用`appendfsync everysec`,当 Redis 后台线程在执行 AOF 文件刷盘时，如果此时磁盘的 IO 负载很高，那这个后台线程在执行刷盘操作（fsync系统调用）时就会被阻塞住。此时的主线程依旧会接收写请求，紧接着，主线程又需要把数据写到文件内存中（write 系统调用），**但此时的后台子线程由于磁盘负载过高，导致 fsync 发生阻塞，迟迟不能返回，那主线程在执行 write 系统调用时，也会被阻塞住**，直到后台线程 fsync 执行完成后，主线程执行 write 才能成功返回。

那什么情况下会导致磁盘 IO 负载过大？

- 子进程正在执行 AOF rewrite，这个过程会占用大量的磁盘 IO 资源
- 有其他应用程序在执行大量的写文件操作，也会占用磁盘 IO 资源

对于情况1，Redis 提供了一个配置项，当子进程在 AOF rewrite 期间，可以让后台子线程不执行刷盘（不触发 fsync 系统调用）操作。这相当于在 AOF rewrite 期间，临时把 appendfsync 设置为了 none，配置如下：

```
# AOF rewrite 期间，AOF 后台子线程不进行刷盘操作，相当于在这期间，临时把 appendfsync 设置为了 none
no-appendfsync-on-rewrite yes
```

当然，开启这个配置项，在 AOF rewrite 期间，如果实例发生宕机，那么此时会丢失更多的数据，性能和数据安全性，你需要权衡后进行选择。

如果占用磁盘资源的是其他应用程序，就需要定位到是哪个应用程序在大量写磁盘，然后把这个应用程序迁移到其他机器上执行就好了，避免对 Redis 产生影响。

AOF基本上是通过两个系统间的调用来完成工作的。 一个是写，用来写数据到AOF， 另外一个是文件数据同步，通过清除硬盘上空核心文件的缓冲来保证用户指定的持久级别。

包括写和文件数据同步的调用都可以导致延迟的根源。 写实例可以阻塞系统范围的同步操作，也可以阻塞当输出的缓冲区满并且内核需要清空到硬盘来接受新的写入的操作。

如果你想诊断AOF相关的延迟原因可以使用strace 命令：

```
sudo strace -p $(pidof redis-server) -T -e trace=fdatasync
```

上面的命令会展示redis主线程里所有的fdatasync系统调用。不包括后台线程执行的fdatasync 调用。如果appendfsync 配置为everysec，则给strace增加-f选项。

用下面命令可以看到fdatasync和write调用：

```
sudo strace -p $(pidof redis-server) -T -e trace=fdatasync,write
```

不过因为write也会向客户端写数据，所以用上面的命令很可能会获得许多与磁盘I/O没有关系的结果。似乎没有办法让strace 只显示慢系统调用，所以要用下面的命令：

```
sudo strace -f -p $(pidof redis-server) -T -e trace=fdatasync,write 2>&1 | grep -v '0.0' | grep -v unfinished
```

## 3.9. 绑定CPU

而 Redis Server 除了主线程服务客户端请求之外，还会创建子进程、子线程。其中子进程用于数据持久化，而子线程用于执行一些比较耗时操作，例如异步释放 fd、异步 AOF 刷盘、异步 lazy-free 等等。如果你把 Redis 进程只绑定了一个 CPU 逻辑核心上，那么当 Redis 在进行数据持久化时，fork 出的子进程会继承父进程的 CPU 使用偏好。

**而此时的子进程会消耗大量的 CPU 资源进行数据持久化（把实例数据全部扫描出来需要耗费CPU），这就会导致子进程会与主进程发生 CPU 争抢，进而影响到主进程服务客户端请求，访问延迟变大。**

我们可以不要让 Redis 进程只绑定在一个 CPU 逻辑核上，而是绑定在多个逻辑核心上，而且，绑定的多个逻辑核心最好是同一个物理核心，这样它们还可以共用 L1/L2 Cache。当然，即便我们把 Redis 绑定在多个逻辑核心上，也只能在一定程度上缓解主线程、子进程、后台线程在 CPU 资源上的竞争。因为这些子进程、子线程还是会在这多个逻辑核心上进行切换，存在性能损耗。

Redis 在 6.0 版本已经推出了这个功能，我们可以通过以下配置，对主线程、后台线程、后台 RDB 进程、AOF rewrite 进程，绑定固定的 CPU 逻辑核心：

```
# Redis Server 和 IO 线程绑定到 CPU核心 0,2,4,6
server_cpulist 0-7:2

# 后台子线程绑定到 CPU核心 1,3
bio_cpulist 1,3

# 后台 AOF rewrite 进程绑定到 CPU 核心 8,9,10,11
aof_rewrite_cpulist 8-11

# 后台 RDB 进程绑定到 CPU 核心 1,10,11
# bgsave_cpulist 1,10-1
```

## 3.10. swapping (操作系统分页)引起的延迟

Linux (以及其他一些操作系统) 可以把内存页存储在硬盘上，反之也能将存储在硬盘上的内存页再加载进内存，这种机制使得内存能够得到更有效的利用。

如果内存页被系统移到了swap文件里，而这个内存页中的数据恰好又被redis用到了（例如要访问某个存储在内存页中的key），系统就会暂停redis进程直到把需要的页数据重新加载进内存。这个操作因为牵涉到随机I/O，所以很慢，会导致无法预料的延迟。

系统之所以要在内存和硬盘之间置换redis页数据主要因为以下三个原因：

- 系统总是要应对内存不足的压力，因为每个运行的进程都想申请更多的物理内存，而这些申请的内存的数量往往超过了实际拥有的内存。简单来说就是redis使用的内存总是比可用的内存数量更多。
- redis实例的数据，或者部分数据可能就不会被客户端访问，所以系统可以把这部分闲置的数据置换到硬盘上。需要把所有数据都保存在内存中的情况是非常罕见的。
- 一些进程会产生大量的读写I/O。因为文件通常都有缓存，这往往会导致文件缓存不断增加，然后产生交换（swap）。请注意，redis RDB和AOF后台线程都会产生大量文件。

```
# 查看 Redis Swap 使用情况
$ cat /proc/$pid/smaps | egrep '^(Swap|Size)'
```

**建议：**

- 增加机器的内存，让 Redis 有足够的内存可以使用
- 整理内存空间，释放出足够的内存供 Redis 使用，然后释放 Redis 的 Swap，让 Redis 重新使用内存

释放 Redis 的 Swap 过程通常要重启实例，为了避免重启实例对业务的影响，一般会先进行主从切换，然后释放旧主节点的 Swap，重启旧主节点实例，待从库数据同步完成后，再进行主从切换即可。

## 3.11. 碎片整理

Redis 的数据都存储在内存中，当我们的应用程序频繁修改 Redis 中的数据时，就有可能会导致 Redis 产生内存碎片。内存碎片会降低 Redis 的内存使用率，我们可以通过执行 INFO 命令，得到这个实例的内存碎片率：

```
# Memory
used_memory:5709194824
used_memory_human:5.32G
used_memory_rss:8264855552
used_memory_rss_human:7.70G
...
mem_fragmentation_ratio:1.45
```

简单，`mem_fragmentation_ratio = used_memory_rss / used_memory`。

其中 used_memory 表示 Redis 存储数据的内存大小，而 used_memory_rss 表示操作系统实际分配给 Redis 进程的大小。

如果 mem_fragmentation_ratio > 1.5，说明内存碎片率已经超过了 50%，这时我们就需要采取一些措施来降低内存碎片了。

解决的方案一般如下：

1. 如果你使用的是 Redis 4.0 以下版本，只能通过重启实例来解决
2. 如果你使用的是 Redis 4.0 版本，它正好提供了自动碎片整理的功能，可以通过配置开启碎片自动整理

**但是，开启内存碎片整理，它也有可能会导致 Redis 性能下降。**

原因在于，Redis 的碎片整理工作是也在**主线程**中执行的，当其进行碎片整理时，必然会消耗 CPU 资源，产生更多的耗时，从而影响到客户端的请求。所以，当你需要开启这个功能时，最好提前测试评估它对 Redis 的影响。

```
# 开启自动内存碎片整理（总开关）
activedefrag yes

# 内存使用 100MB 以下，不进行碎片整理
active-defrag-ignore-bytes 100mb

# 内存碎片率超过 10%，开始碎片整理
active-defrag-threshold-lower 10
# 内存碎片率超过 100%，尽最大努力碎片整理
active-defrag-threshold-upper 100

# 内存碎片整理占用 CPU 资源最小百分比
active-defrag-cycle-min 1
# 内存碎片整理占用 CPU 资源最大百分比
active-defrag-cycle-max 25

# 碎片整理期间，对于 List/Set/Hash/ZSet 类型元素一次 Scan 的数量
active-defrag-max-scan-fields 1000
```

我们需要结合 Redis 机器的负载情况，以及应用程序可接受的延迟范围进行评估，合理调整碎片整理的参数，尽可能降低碎片整理期间对 Redis 的影响。

## 3.12. 网络带宽过载

如果以上产生性能问题的场景，你都规避掉了，而且 Redis 也稳定运行了很长时间，但在某个时间点之后开始，操作 Redis 突然开始变慢了，而且一直持续下去，这种情况又是什么原因导致？

此时你需要排查一下 Redis 机器的网络带宽是否过载，是否存在某个实例把整个机器的网路带宽占满的情况。

网络带宽过载的情况下，服务器在 TCP 层和网络层就会出现数据包发送延迟、丢包等情况。

Redis 的高性能，除了操作内存之外，就在于网络 IO 了，如果网络 IO 存在瓶颈，那么也会严重影响 Redis 的性能。

如果确实出现这种情况，你需要及时确认占满网络带宽 Redis 实例，如果属于正常的业务访问，那就需要及时扩容或迁移实例了，避免因为这个实例流量过大，影响这个机器的其他实例。

运维层面，你需要对 Redis 机器的各项指标增加监控，包括网络流量，在网络流量达到一定阈值时提前报警，及时确认和扩容。

# 4. 参考资料

http://www.oschina.net/translate/redis-latency-problems-troubleshooting

https://zhongfox.github.io/2016/01/31/redis-latency/

https://mp.weixin.qq.com/s/gYQn9tdFK9tHJDoMWcU4cQ