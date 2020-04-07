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
## 计算延迟时间
如果你正在经历响应延迟问题，你或许能够根据应用程序的具体情况算出它的延迟响应时间，或者你的延迟问题非常明显，宏观看来，一目了然。不管怎样吧，用redis-cli可以算出一台Redis 服务器的到底延迟了多少毫秒。

	redis-cli --latency -h `host` -p `port`

## 延迟配置
开启延迟监控

	CONFIG SET latency-monitor-threshold 100 单位为毫秒

默认情况latency-monitor-threshold为0, 即延迟监控是关闭的
延迟监控功能占用内存很小, 不过对于性能良好的redis也没有必要开启

## 延迟原因
###  网络通信延迟
当用户连接到Redis通过TCP/IP连接或Unix域连接，千兆网络的典型延迟大概200us，而Unix域socket可能低到30us。这完全基于你的网络和系统硬件。在通信本身之上，系统增加了更多的延迟（线程调度，CPU缓存，NUMA替换等等）。系统引起的延迟在虚拟机环境远远高于在物理机器环境。 
是即使Redis处理大多数命令在微秒之下，客户机和服务器之间的交互也必然消耗系统相关的延迟。 
**建议**

- 尽可能的使用物理机而不是虚拟机来做服务器
- 不要经常的connect/disconnect与服务器的连接（尤其是对基于web的应用），尽可能的延长与服务器连接的时间。（长连接）
- 如果你的客户端和服务器在同一台主机上，则使用Unix域套接字
- 优先使用聚合命令(MSET/MGET), 而不是pipeline
- 优先使用pipeline, 而不是频繁发送命令(多次网络往返)
- 对不适合使用pipeline的命令, 可以考虑使用lua脚本

### 单线程
Redis 使用了单线程的处理所有的客户端请求。当一个请求执行得很慢，其他的客户端调用就必须等待这个请求执行完毕。GET or SET or LPUSH 等命令执行时间是常数, 不过类似 SORT, LREM, SUNION等操作多个元素的命令执行时间是O(N)

- 减少多元素慢命令的使用
-  对于 KEYS命令只能用于线下调试, 生产环境可以使用 SCAN, SSCAN, HSCAN and ZSCAN等命令代替
- 使用主从复制, 将慢的命令放到复制机器上执行
- 使用SLOWLOG 分析慢查询

### fork 引起的延迟
生成RDB或者AOF会使redis 主线程fork后台线程, 这会造成一定延迟。

对于运行在一个linux/AMD64系统上的实例来说，内存会按照每页4KB的大小分页。为了实现虚拟地址到物理地址的转换，每一个进程将会存储一个分页表（树状形式表现），分页表将至少包含一个指向该进程地址空间的指针。所以一个空间大小为24GB的redis实例，需要的分页表大小为  24GB/4KB*8 = 48MB。

当一个后台的save命令执行时，实例会启动新的线程去申请和拷贝48MB的内存空间。这将消耗一些时间和CPU资源，尤其是在虚拟机上申请和初始化大块内存空间时，消耗更加明显。

不同系统中的Fork时间

    Linux beefy VM on VMware 6.0GB RSS forked 77 微秒 (每GB 12.8 微秒 ).
    Linux running on physical machine (Unknown HW) 6.1GB RSS forked 80 微秒(每GB 13.1微秒)
    Linux running on physical machine (Xeon @ 2.27Ghz) 6.9GB RSS forked into 62 微秒 (每GB 9 微秒).
    Linux VM on 6sync (KVM) 360 MB RSS forked in 8.2 微秒 (每GB 23.3 微秒).
    Linux VM on EC2 (Xen) 6.1GB RSS forked in 1460 微秒 (每GB 239.3 微秒).
    Linux VM on Linode (Xen) 0.9GBRSS forked into 382 微秒 (每GB 424 微秒).


### swapping (操作系统分页)引起的延迟
Linux (以及其他一些操作系统) 可以把内存页存储在硬盘上，反之也能将存储在硬盘上的内存页再加载进内存，这种机制使得内存能够得到更有效的利用。

如果内存页被系统移到了swap文件里，而这个内存页中的数据恰好又被redis用到了（例如要访问某个存储在内存页中的key），系统就会暂停redis进程直到把需要的页数据重新加载进内存。这个操作因为牵涉到随机I/O，所以很慢，会导致无法预料的延迟。

系统之所以要在内存和硬盘之间置换redis页数据主要因为以下三个原因：

    系统总是要应对内存不足的压力，因为每个运行的进程都想申请更多的物理内存，而这些申请的内存的数量往往超过了实际拥有的内存。简单来说就是redis使用的内存总是比可用的内存数量更多。
    redis实例的数据，或者部分数据可能就不会被客户端访问，所以系统可以把这部分闲置的数据置换到硬盘上。需要把所有数据都保存在内存中的情况是非常罕见的。
    一些进程会产生大量的读写I/O。因为文件通常都有缓存，这往往会导致文件缓存不断增加，然后产生交换（swap）。请注意，redis RDB和AOF后台线程都会产生大量文件。

所幸Linux提供了很好的工具来诊断这个问题，所以当延迟疑似是swap引起的，最简单的办法就是使用Linux提供的工具去确诊。

### AOF 和硬盘I/O操作延迟

AOF基本上是通过两个系统间的调用来完成工作的。 一个是写，用来写数据到AOF， 另外一个是文件数据同步，通过清除硬盘上空核心文件的缓冲来保证用户指定的持久级别。

包括写和文件数据同步的调用都可以导致延迟的根源。 写实例可以阻塞系统范围的同步操作，也可以阻塞当输出的缓冲区满并且内核需要清空到硬盘来接受新的写入的操作。
如果你想诊断AOF相关的延迟原因可以使用strace 命令：

	sudo strace -p $(pidof redis-server) -T -e trace=fdatasync

上面的命令会展示redis主线程里所有的fdatasync系统调用。不包括后台线程执行的fdatasync 调用。如果appendfsync 配置为everysec，则给strace增加-f选项。
用下面命令可以看到fdatasync和write调用：

	sudo strace -p $(pidof redis-server) -T -e trace=fdatasync,write

不过因为write也会向客户端写数据，所以用上面的命令很可能会获得许多与磁盘I/O没有关系的结果。似乎没有办法让strace 只显示慢系统调用，所以要用下面的命令：

	sudo strace -f -p $(pidof redis-server) -T -e trace=fdatasync,write 2>&1 | grep -v '0.0' | grep -v unfinished

### 数据过期造成的延迟
redis有两种方式来去除过期的key：

- lazy 方式，在key被请求的时候才检查是否过期。 to be already expired.
- active 方式，每0.1秒进行一次过期检查。

active过期模式是自适应的，每过100毫秒开始一次过期检查（每秒10次），每次作如下操作：

- 根据 REDIS_EXPIRELOOKUPS_PER_CRON 的值去除已经过期的key（是指如果过期的key数量超过了REDIS_EXPIRELOOKUPS_PER_CRON 的值才会启动过期操作，太少就不必了。这个值默认为10）, evicting all the keys already expired.
- 假如超过25%（是指REDIS_EXPIRELOOKUPS_PER_CRON这个值的25%，这个值默认为10，译者注）的key已经过期，则重复一遍检查失效的过程。

REDIS_EXPIRELOOKUPS_PER_CRON 默认为10， 过期检查一秒会执行10次，通常在actively模式下1秒能处理100个key。在过期的key有一段时间没被访问的情况下这个清理速度已经足够了，所以 lazy模式基本上没什么用。1秒只过期100个key也不会对redis造成多大的影响。

这种算法式是自适应的，如果发现有超过指定数量25%的key已经过期就会循环执行。这个过程每秒会运行10次，这意味着随机样本中超过25%的key会在1秒内过期。

通常来讲如果有很多key在同一秒过期，超过了所有key的25%，redis就会阻塞直到过期key的比例下降到25%以下

**大量key同时过期会对系统延迟造成影响。 **

# 参考资料


http://www.oschina.net/translate/redis-latency-problems-troubleshooting
https://zhongfox.github.io/2016/01/31/redis-latency/
