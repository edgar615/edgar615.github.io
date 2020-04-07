---
layout: post
title: redis主从复制
date: 2020-04-07
categories:
    - redis
comments: true
permalink: redis-slave.html
---

主从复制，是指将一台 Redis 服务器的数据，复制到其他的 Redis 服务器。前者称为主节点(master)，后者称为从节点(slave)；数据的复制是单向的，只能由主节点到从节点。

默认情况下，每台 Redis 服务器都是主节点；且一个主节点可以有多个从节点(或没有从节点)，但一个从节点只能有一个主节点。

主从复制的作用主要包括：

- **数据冗余：**主从复制实现了数据的热备份，是持久化之外的一种数据冗余方式。
- **故障恢复：**当主节点出现问题时，可以由从节点提供服务，实现快速的故障恢复；实际上是一种服务的冗余。
- **负载均衡：**在主从复制的基础上，配合读写分离，可以由主节点提供写服务，由从节点提供读服务（即写  Redis 数据时应用连接主节点，读 Redis  数据时应用连接从节点），分担服务器负载；尤其是在写少读多的场景下，通过多个从节点分担读负载，可以大大提高 Redis 服务器的并发量。
- **高可用基石：**除了上述作用以外，主从复制还是哨兵和集群能够实施的基础，因此说主从复制是 Redis 高可用的基础。

# 使用主从复制

redis有三种方式建立主从复制

1. 在从节点的配置文件上加入`slaveof <masterip> <masterport>`
2. 在从节点的redis-server启动命令后加上 `./redis-server --slaveof <masterip> <masterport>`
3. 在从节点上使用命令 `slaveof <masterip> <masterport>`

示例
**启动主节点**

```
src/redis-server 
```

**启动从节点**，注意避免使用主节点的持久化文件加载

```
src/redis-server --port 6380 
```

**主节点上有一个key**

```
127.0.0.1:6379> DBSIZE
(integer) 1
127.0.0.1:6379> keys *
1) "foo"
```

**建立主从复制之前，从库没有KEY**

```
127.0.0.1:6380> dbsize
(integer) 0
```

**建立主从复制**

```
127.0.0.1:6380> slaveof 127.0.0.1 6379
OK
```

**主节点的日志如下**

```
27984:M 21 Nov 16:42:44.889 * Slave 127.0.0.1:6380 asks for synchronization
27984:M 21 Nov 16:42:44.890 * Partial resynchronization not accepted: Replication ID mismatch (Slave asked for 'b83d19387d13d6820a8190fb037b121dc80697e6', my replication IDs are 'e330daced60ae878cffc7a7e02ccb54c11d73dfe' and '0000000000000000000000000000000000000000')
27984:M 21 Nov 16:42:44.890 * Starting BGSAVE for SYNC with target: disk
27984:M 21 Nov 16:42:44.891 * Background saving started by pid 28075
28075:C 21 Nov 16:42:44.894 * DB saved on disk
28075:C 21 Nov 16:42:44.894 * RDB: 6 MB of memory used by copy-on-write
27984:M 21 Nov 16:42:44.952 * Background saving terminated with success
27984:M 21 Nov 16:42:44.952 * Synchronization with slave 127.0.0.1:6380 succeeded
```

**节点的日志如下**

```
28069:S 21 Nov 16:42:44.345 * Before turning into a slave, using my master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.
28069:S 21 Nov 16:42:44.345 * SLAVE OF 127.0.0.1:6379 enabled (user request from 'id=2 addr=127.0.0.1:36184 fd=7 name= age=91 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=32768 obl=0 oll=0 omem=0 events=r cmd=slaveof')
28069:S 21 Nov 16:42:44.889 * Connecting to MASTER 127.0.0.1:6379
28069:S 21 Nov 16:42:44.889 * MASTER <-> SLAVE sync started
28069:S 21 Nov 16:42:44.889 * Non blocking connect for SYNC fired the event.
28069:S 21 Nov 16:42:44.889 * Master replied to PING, replication can continue...
28069:S 21 Nov 16:42:44.889 * Trying a partial resynchronization (request b83d19387d13d6820a8190fb037b121dc80697e6:1).
28069:S 21 Nov 16:42:44.893 * Full resync from master: d7ffc06a335aa6d77dad778b8b3088d6e1212fbd:0
28069:S 21 Nov 16:42:44.893 * Discarding previously cached master state.
28069:S 21 Nov 16:42:44.952 * MASTER <-> SLAVE sync: receiving 189 bytes from master
28069:S 21 Nov 16:42:44.952 * MASTER <-> SLAVE sync: Flushing old data
28069:S 21 Nov 16:42:44.952 * MASTER <-> SLAVE sync: Loading DB in memory
28069:S 21 Nov 16:42:44.953 * MASTER <-> SLAVE sync: Finished with success
```

**建立主从复制之后，从库有了key**

```
127.0.0.1:6380> dbsize
(integer) 1
127.0.0.1:6380> get foo
"bar"
```

slaveof是一个异步命令，执行slaveof时，节点只保存主节点信息后返回，后续复制流程在节点内部异步执行。

## 查看复制状态

可以使用`info replication`查看复制状态

**主节点**

```
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=127.0.0.1,port=6380,state=online,offset=690,lag=0
master_replid:d7ffc06a335aa6d77dad778b8b3088d6e1212fbd
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:690
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:690
```

**从节点**

```
127.0.0.1:6380> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
master_last_io_seconds_ago:9
master_sync_in_progress:0
slave_repl_offset:606
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:d7ffc06a335aa6d77dad778b8b3088d6e1212fbd
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:606
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:606
```

## 断开复制
在从节点执行`slaveof no one`可以断开与主节点复制关系。从节点断开复制后并不会抛弃原有数据，只是无法再获取主节点上的数据变化

## 切换主 节点
在从节点上通过`slaveof <masterip> <masterport>``命令可以把当前从节点对主节点的复制切换到另一个主节点：
1. 断开与旧主节点复制关系
2. 与新主就节点建立复制关系
3. 删除从节点当前所有数据
4. 对新主节点进行复制操作

# 复制过程

redis的复制过程如下图所示

![](/assets/images/posts/redis-slave/redis-slave-1.jpg)

1. 从节点执行 slaveof 命令。      
2. 从节点只是保存了 slaveof 命令中主节点的信息，并没有立即发起复制。      
3. 从节点内部的定时任务发现有主节点的信息，开始使用 socket 连接主节点。    
4. 连接建立成功后，发送 ping 命令，希望得到 pong 命令响应，否则会进行重连。      
5. 如果主节点设置了权限，那么就需要进行权限验证，如果验证失败，复制终止。        
6. 权限验证通过后，进行数据同步，**这是耗时最长的操作**，主节点将把所有的数据全部发送给从节点。    
7. 当主节点把当前的数据同步给从节点后，便完成了复制的建立流程。接下来，**主节点就会持续的把写命令发送给从节点，保证主从数据一致性**。

## 连接建立阶段
**保存主节点信息**

从节点服务器内部维护了两个字段，即 masterhost 和 masterport 字段，用于存储主节点的 ip 和 port 信息。

需要注意的是，slaveof 是异步命令，从节点完成主节点 ip 和 port 的保存后，向发送 slaveof 命令的客户端直接返回 OK，实际的复制操作在这之后才开始进行。

**建立 Socket 连接**

从节点每秒 1 次调用复制定时函数 replicationCron()，如果发现了有主节点可以连接，便会根据主节点的 ip 和 port，创建 socket 连接。

如果连接成功，则：

- **从节点：**为该 socket 建立一个专门处理复制工作的文件事件处理器，负责后续的复制工作，如接收 RDB 文件、接收命令传播等。
- **主节点：**接收到从节点的 socket 连接后（即 accept 之后），为该 socket 创建相应的客户端状态，并将从节点看做是连接到主节点的一个客户端，后面的步骤会以从节点向主节点发送命令请求的形式来进行。

**发送 Ping 命令**

从节点成为主节点的客户端之后，发送 ping 命令进行首次请求，目的是：检查 socket 连接是否可用，以及主节点当前是否能够处理请求。

从节点发送 ping 命令后，可能出现 3 种情况：

- **返回pong：**说明 socket 连接正常，且主节点当前可以处理请求，复制过程继续。
- **超时：**一定时间后从节点仍未收到主节点的回复，说明 socket 连接不可用，则从节点断开 socket 连接，并重连。
- **返回 pong 以外的结果：**如果主节点返回其他结果，如正在处理超时运行的脚本，说明主节点当前无法处理命令，则从节点断开 socket 连接，并重连。

**身份验证**

如果从节点中设置了 masterauth 选项，则从节点需要向主节点进行身份验证；没有设置该选项，则不需要验证。

从节点进行身份验证是通过向主节点发送 auth 命令进行的，auth 命令的参数即为配置文件中的 masterauth 的值。

如果主节点设置密码的状态，与从节点 masterauth 的状态一致（一致是指都存在，且密码相同，或者都不存在），则身份验证通过，复制过程继续；如果不一致，则从节点断开 socket 连接，并重连。

**发送从节点端口信息**

身份验证之后，从节点会向主节点发送其监听的端口号（前述例子中为 6380），主节点将该信息保存到该从节点对应的客户端的 slave_listening_port 字段中。

该端口信息除了在主节点中执行 info Replication 时显示以外，没有其他作用。

## 数据同步阶段
主从节点之间的连接建立以后，便可以开始进行数据同步，该阶段可以理解为从节点数据的初始化。

数据同步阶段是主从复制最核心的阶段，根据主从节点当前状态的不同，可以分为全量复制和部分复制。

需要注意的是，在数据同步阶段之前，从节点是主节点的客户端，主节点不是从节点的客户端；而到了这一阶段及以后，主从节点互为客户端。

原因在于：在此之前，主节点只需要响应从节点的请求即可，不需要主动发请求，而在数据同步阶段和后面的命令传播阶段，主节点需要主动向从节点发送请求（如推送缓冲区中的写命令），才能完成复制。

redis 同步有 2 个命令：sync 和 psync，前者是 redis 2.8 之前的同步命令，后者是 redis 2.8 为了优化 sync 新设计的命令。我们会重点关注 2.8 的 psync 命令。

## psync

在redis内部有一条命令`psync`，是做同步的命令，它可以完成全量复制和部分复制的功能，当启动slave节点时，它会发送`psync`命令给主节点，需要传递两个参数，`runid`和`offset`(偏移量)，也就是从向主传递主节点的runid以及自己的偏移量。

psync 执行流程：

![](/assets/images/posts/redis-slave/redis-slave-2.jpg)

从节点发送 psync 命令给主节点，runId 就是目标主节点的 ID，如果没有默认为 -1，offset 是从节点保存的复制偏移量，如果是第一次复制则为 -1.

主节点会根据 runid 和 offset 决定返回结果：

1. 如果回复 +FULLRESYNC {runId} {offset} ，那么从节点将触发全量复制流程。      
2. 如果回复 +CONTINUE，从节点将触发部分复制。        
3. 如果回复 +ERR，说明主节点不支持 2.8 的 psync 命令，将使用 sync 执行全量复制。

**psync 命令需要 3 个组件支持**：

1. 主从节点各自复制偏移量
2. 主节点复制积压缓冲区
3. 主节点运行 ID

### 主从节点各自复制偏移量：

1. 参与复制的主从节点都会维护自身的复制偏移量。 
2. 主节点在处理完写入命令后，会把命令的字节长度做累加记录，统计信息在 info replication 中的 masterreploffset 指标中。
3. 从节点每秒钟上报自身的的复制偏移量给主节点，因此主节点也会保存从节点的复制偏移量。
4. 从节点在接收到主节点发送的命令后，也会累加自身的偏移量，统计信息在 info replication 中。
5. 通过对比主从节点的复制偏移量，可以判断主从节点数据是否一致。

启动主从redis，并在写入命令执行前后查看主偏移量：`redis-cli -p 6379 info replication | grep master_repl`

### 主节点复制积压缓冲区：

1. 复制积压缓冲区是一个保存在主节点的一个固定长度的先进先出的队列，默认大小 1MB。     
2. 这个队列在 slave 连接时创建。这时主节点响应写命令时，不但会把命令发送给从节点，也会写入复制缓冲区。        
3. 他的作用就是用于部分复制和复制命令丢失的数据补救。通过 info replication 可以看到相关信息。

### 主节点运行 ID：

1. 每个 redis 启动的时候，都会生成一个 40 位的运行 ID，称为`runid`。        
2. 运行 ID 的主要作用是用来识别 Redis 节点。如果使用 ip+port 的方式，那么如果主节点重启修改了 RDB/AOF 数据，从节点再基于偏移量进行复制将是不安全的。所以，当运行 id 变化后，从节点将进行全量复制。也就是说，redis 重启后，默认从节点会进行全量复制。

查看runid：`redis-cli -p 6379 info | grep run`

当主从复制在初次复制时，主节点将自己的runid发送给从节点，从节点将这个runid保存起来,当断线重连时，从节点会将这个runid发送给主节点。主节点根据runid判断能否进行部分复制：

- 如果从节点保存的runid与主节点现在的runid相同，说明主从节点之前同步过，主节点会更具offset偏移量之后的数据判断是否执行部分复制，如果offset偏移量之后的数据仍然都在复制积压缓冲区里，则执行部分复制，否则执行全量复制；
- 如果从节点保存的runid与主节点现在的runid不同，说明从节点在断线前同步的redis节点并不是当前的主节点，只能进行全量复制;

## 命令传播阶段

数据同步阶段完成后，主从节点进入命令传播阶段；在这个阶段主节点将自己执行的写命令发送给从节点，从节点接收命令并执行，从而保证主从节点数据的一致性。

在命令传播阶段，除了发送写命令，主从节点还维持着心跳机制：PING 和 REPLCONF ACK。

由于心跳机制的原理涉及部分复制，因此将在介绍了部分复制的相关内容后单独介绍该心跳机制。

**延迟与不一致**：需要注意的是，命令传播是异步的过程，即主节点发送写命令后并不会等待从节点的回复；因此实际上主从节点之间很难保持实时的一致性，延迟在所难免。

数据不一致的程度，与主从节点之间的网络状况、主节点写命令的执行频率、以及主节点中的 repl-disable-tcp-nodelay 配置等有关。

repl-disable-tcp-nodelay no：该配置作用于命令传播阶段，控制主节点是否禁止与从节点的 TCP_NODELAY；默认 no，即不禁止 TCP_NODELAY。

- 当设置为 yes 时，TCP 会对包进行合并从而减少带宽，但是发送的频率会降低，从节点数据延迟增加，一致性变差；具体发送频率与 Linux 内核的配置有关，默认配置为 40ms。
- 当设置为 no 时，TCP 会立马将主节点的数据发送给从节点，带宽增加但延迟变小。

一般来说，只有当应用对 Redis 数据不一致的容忍度较高，且主从节点之间网络状况不好时，才会设置为 yes；多数情况使用默认值 no。

# 数据同步阶段：全量复制和部分复制

在 Redis 2.8 以前，从节点向主节点发送 sync 命令请求同步数据，此时的同步方式是全量复制。

在 Redis 2.8 及以后，从节点可以发送 psync 命令请求同步数据，此时根据主从节点当前状态的不同，同步方式可能是全量复制或部分复制。后文介绍以 Redis 2.8 及以后版本为例。

**全量复制：**用于初次复制或其他无法进行部分复制的情况，将主节点中的所有数据都发送给从节点，是一个非常重型的操作。

**部分复制：**用于网络中断等情况后的复制，只将中断期间主节点执行的写命令发送给从节点，与全量复制相比更加高效。

需要注意的是，如果网络中断时间过长，导致主节点没有能够完整地保存中断期间执行的写命令，则无法进行部分复制，仍使用全量复制。

## 全量复制

全量复制主节点会将RDB文件也就是当前状态去同步给slave，在此期间主新写入的命令会单独记录起来，然后当RDB文件加载完毕之后，会通过偏移量对比将这个期间产生的写入值同步给slave，这样就能达到数据完全同步的效果

![](/assets/images/posts/redis-slave/redis-slave-3.jpg)

1. Redis内部会发出一个同步命令，刚开始是Psync命令，Psync ? -1表示要求master主机同步数据
2. 主机会向从机发送run_id和offset，因为slave并没有对应的 offset，所以是全量复制
3. 从机slave会保存主机master的基本信息
4. 主节点收到全量复制的命令后，执行bgsave（异步执行），在后台生成RDB文件（快照），并使用一个缓冲区（称为复制缓冲区）记录从现在开始执行的所有写命令（如果从节点花费时间过长，将导致缓冲区溢出，最后全量同步失败） 
5. 主机发送RDB文件给从机
6. 发送缓冲区数据
7. 刷新旧的数据。从节点在载入主节点的数据之前要先将老数据清除
8. 加载RDB文件将数据库状态更新至主节点执行bgsave时的数据库状态和缓冲区数据的加载。

### 全量复制的开销

实际上全量复制的开销是非常大的，主要体现在如下方面

- bgsave时间(对cpu、 内存、硬盘都会有一定的开销)
- RDB文件网络传输时间(网络带宽)
- 从节点清空数据时间(根据从节点的数据规模)
- 从节点加载RDB的时间
- 可能的AOF重写时间(在最后从加载完RDB之后如果开启了AOF，会做AOF重写)

全量复制除了上述开销之外，还会有个问题：
**假如master和slave网络发生了抖动，那一段时间内这些数据就会丢失，对于slave来说这段时间master更新的数据是不知道的**。最简单的方式就是再做一次全量复制，从而获取到最新的数据，在redis2.8之前是这么做的。

## 部分复制

部分复制是Redis 2.8以后出现的，用于处理在主从复制中因网络闪断等原因造成的数据丢失场景，当从节点再次连上主节点后，如果条件允许，主节点会补发丢失数据给从节点。因为补发的数据远远小于全量数据，可以有效避免全量复制的过高开销，需要注意的是，如果网络中断时间过长，造成主节点没有能够完整地保存中断期间执行的写命令，则无法进行部分复制，仍使用全量复制

![](/assets/images/posts/redis-slave/redis-slave-4.png)

1. 如果网络抖动（连接断开 connection lost）
2. 主机master 还是会写 repl_back_buffer（复制缓冲区）
3. 从机slave 会继续尝试连接主机
4. 从机slave 会把自己当前 run_id 和偏移量传输给主机 master，并且执行 pysnc 命令同步
5. 如果master发现你的偏移量是在缓冲区的范围内，就会返回 continue命令（不在期间内就证明你已经错过了很多数据，buffer也是有限的，默认是1M）
6. 同步了offset的部分数据，所以部分复制的基础就是偏移量 offset。

部分复制的实现，依赖于三个重要的概念：

**复制偏移量：**
主节点和从节点分别维护一个复制偏移量（offset），代表的是主节点向从节点传递的字节数。

主节点每次向从节点传播 N 个字节数据时，主节点的 offset 增加 N；从节点每次收到主节点传来的 N 个字节数据时，从节点的 offset 增加 N。

offset 用于判断主从节点的数据库状态是否一致：如果二者 offset 相同，则一致；如果 offset 不同，则不一致，此时可以根据两个 offset 找出从节点缺少的那部分数据。

例如，如果主节点的 offset 是 1000，而从节点的 offset 是 500，那么部分复制就需要将 offset 为 501-1000 的数据传递给从节点。

而 offset 为 501-1000 的数据存储的位置，就是下面要介绍的复制积压缓冲区。

**复制积压缓冲区：**
复制积压缓冲区是由主节点维护的、固定长度的、先进先出(FIFO)队列，默认大小 1MB。

当主节点开始有从节点时创建，其作用是备份主节点最近发送给从节点的数据。注意，**无论主节点有一个还是多个从节点，都只需要一个复制积压缓冲区。**

在命令传播阶段，主节点除了将写命令发送给从节点，还会发送一份给复制积压缓冲区，作为写命令的备份；除了存储写命令，复制积压缓冲区中还存储了其中的每个字节对应的复制偏移量（offset）。

由于复制积压缓冲区定长且是先进先出，所以它保存的是主节点最近执行的写命令；时间较早的写命令会被挤出缓冲区。

由于该缓冲区长度固定且有限，因此可以备份的写命令也有限，当主从节点 offset 的差距过大超过缓冲区长度时，将无法执行部分复制，只能执行全量复制。

反过来说，为了提高网络中断时部分复制执行的概率，可以根据需要增大复制积压缓冲区的大小(通过配置repl-backlog-size)。

例如如果网络中断的平均时间是 60s，而主节点平均每秒产生的写命令(特定协议格式)所占的字节数为 100KB，则复制积压缓冲区的平均需求为 6MB。

保险起见，可以设置为 12MB，来保证绝大多数断线情况都可以使用部分复制。

从节点将 offset 发送给主节点后，主节点根据 offset 和缓冲区大小决定能否执行部分复制：

- 如果 offset 偏移量之后的数据，仍然都在复制积压缓冲区里，则执行部分复制。
- 如果 offset 偏移量之后的数据已不在复制积压缓冲区中（数据已被挤出），则执行全量复制。

**服务器运行 ID(runid)：**每个 Redis 节点(无论主从)，在启动时都会自动生成一个随机 ID(每次启动都不一样)，由 40 个随机的十六进制字符组成；runid 用来唯一识别一个 Redis 节点。

通过` info Server` 命令，可以查看节点的 runid：

主从节点初次复制时，主节点将自己的 runid 发送给从节点，从节点将这个 runid 保存起来；当断线重连时，从节点会将这个 runid 发送给主节点。

主节点根据 runid 判断能否进行部分复制：

- 如果从节点保存的 runid 与主节点现在的 runid 相同，说明主从节点之前同步过，主节点会继续尝试使用部分复制(到底能不能部分复制还要看 offset 和复制积压缓冲区的情况)。
- 如果从节点保存的 runid 与主节点现在的 runid 不同，说明从节点在断线前同步的 Redis 节点并不是当前的主节点，只能进行全量复制。

# 心跳

主从节点在建立复制后，他们之间维护着长连接并彼此发送心跳命令。

心跳的关键机制如下：

1. 主从都有心跳检测机制，各自模拟成对方的客户端进行通信，通过 client list 命令查看复制相关客户端信息，主节点的连接状态为 flags = M，从节点的连接状态是 flags = S。 
2. 主节点默认每隔 10 秒对从节点发送 ping 命令，可修改配置 repl-ping-slave-period 控制发送频率。
3. 从节点在主线程每隔一秒发送 replconf ack{offset} 命令，给主节点上报自身当前的复制偏移量。
4. 主节点收到 replconf 信息后，判断从节点超时时间，如果超过 repl-timeout 60 秒，则判断节点下线。

![](/assets/images/posts/redis-slave/redis-slave-5.jpg)

为了降低主从延迟，一般把 redis 主从节点部署在相同的机房/同城机房，避免网络延迟带来的网络分区造成的心跳中断等情况。

**REPLCONF ACK 命令的作用包括：**

**实时监测主从节点网络状态**：该命令会被主节点用于复制超时的判断。此外，在主节点中使用 info Replication，可以看到其从节点的状态中的 lag 值，代表的是主节点上次收到该 REPLCONF ACK 命令的时间间隔。

在正常情况下，该值应该是 0 或 1 `info Replication | grep lag`

**检测命令丢失**：从节点发送了自身的 offset，主节点会与自己的 offset 对比，如果从节点数据缺失（如网络丢包），主节点会推送缺失的数据（这里也会利用复制积压缓冲区）。

注意，offset 和复制积压缓冲区，不仅可以用于部分复制，也可以用于处理命令丢失等情形；区别在于前者是在断线重连后进行的，而后者是在主从节点没有断线的情况下进行的

**辅助保证从节点的数量和延迟**：Redis 主节点中使用 min-slaves-to-write 和 min-slaves-max-lag 参数，来保证主节点在不安全的情况下不会执行写命令；所谓不安全，是指从节点数量太少，或延迟过高。

例如 `min-slaves-to-write` 和` min-slaves-max-lag` 分别是 3 和 10，含义是如果从节点数量小于 3 个，或所有从节点的延迟值都大于 10s，则主节点拒绝执行写命令。

而这里从节点延迟值的获取，就是通过主节点接收到 REPLCONF ACK 命令的时间来判断的，即前面所说的 info Replication 中的 lag 值。

# 异步复制

主节点不但负责数据读写，还负责把写命令同步给从节点，写命令的发送过程是异步完成，也就是说主节点处理完写命令后立即返回客户度，并不等待从节点复制完成。

异步复制的步骤很简单，如下：

1. 主节点接受处理命令。
2. 主节点处理完后返回响应结果 。
3. 对于修改命令，异步发送给从节点，从节点在主线程中执行复制的命令。

![](/assets/images/posts/redis-slave/redis-slave-6.jpg)

# redis-4.0.x中如何解决redis重启runid变化引起的全量复制

在 2.8 版本之前 redis 没有增量同步的功能，主从只要重连就必须全量同步数据。如果实例数据量比较大的情况下，网络轻轻一抖就会把主从的网卡跑满从而影响正常服务。2.8 为了解决这个问题引入了 PSYNC (partial sync)功能，顾名思义就是增量同步。

从库尝试发送 PSYNC 命令到主库，而不是直接使用 SYNC命令进行全量同步 
主库判断是否满足 PSYNC 条件, 满足就返回 +CONTINUE 进行增量同步, 否则返回 +FULLRESYNC runid offfset进行全量同步。

redis 判断是否允许 psync 有两个条件:

    条件一: psync 命令携带的 runid 需要和主库的 runid 一致才可以进行增量同步，否则需要全量同步。 
    条件二: psync 命令携带的 offset 是否超过缓冲区（repl-backlog-size，默认1M）。如果超过则需要全量同步，否则就进行增量同步。

虽然 2.8 引入的 psync 可以解决短时间主从同步断掉重连问题，但以下几个场景仍然是需要全量同步:

    主从断开时间过长，超出了缓冲区覆盖范围（这个可以通过修改参数规避） 
    主库/从库有重启过。因为 runnid 重启后就会丢失，所以当前机制无法做增量同步。 
    从库提升为主库。其他从库切到新主库全部要全量不同数据，因为新主库的 runnid 跟老的主库是不一样的。

为了解决主从角色切换导致的重新全量同步，redis4.0新版本除了增加混合持久化，还优化了psync，在原psync基础上新增两个复制id：

- master_replid : 复制ID1(后文简称：replid1)，一个长度为41个字节(40个随机串+'\0')的字符串。redis实例都有，和runid没有直接关联，但和runid生成规则相同，都是由getRandomHexChars函数生成。当实例变为从实例后，自己的replid1会被主实例的replid1覆盖。

- master_replid2：复制id2(后文简称:replid2),默认初始化为全0，用于存储上次主实例的replid1。

实例的replid信息，可通过info replication进行查看； 示例如下：

```
127.0.0.1:6385> info replication
# Replication
role:slave
master_host:xxxx    
master_port:6382
master_link_status:up
slave_repl_offset:119750
master_replid:fe093add4ab71544ce6508d2e0bf1dd0b7d1c5b2  //这里是主实例的replid1相同
master_replid2:0000000000000000000000000000000000000000  //未发生切换，即主实例未发生过变化，所以是初始值全"0"
master_repl_offset:119750
second_repl_offset:-1

```

在4.0之前的版本，redis复制信息完全丢失，所以每个实例重启后只能进行全量复制，到了4.0版本，主要解决了两种情况下不能进行增量复制的问题:

## 重启

在之前的版本，redis重启后，复制信息是完全丢失;所以从实例重启后，只能进行全量重新同步。
 redis4.0为实现重启后，仍可进行部分重新同步，主要做以下3点：

1. redis关闭时，把复制信息作为辅助字段(AUX Fields)存储在RDB文件中；以实现同步信息持久化。
2. redis启动加载RDB文件时，会把复制信息赋给相关字段；为部分同步
3. redis重新同步时，会上报repl-id和repl-offset同步信息，如果和主实例匹配，且offset还在主实例的复制积压缓冲区内，则只进行部分重新同步。

- redis关闭时，持久化复制信息到RDB

redis在关闭时，通过shutdown save,都会调用rdbSaveInfoAuxFields函数， 把当前实例的repl-id和repl-offset保存到RDB文件中。
说明：当前的RDB存储的数据内容和复制信息是一致性的。

生成的RDB文件，可以通过redis自带的redis-check-rdb工具查看辅助字段信息。其中repl两字段信息和info中的相同；

```
$shell> /src/redis-check-rdb  dump.rdb      
[offset 0] Checking RDB file dump.rdb
[offset 26] AUX FIELD redis-ver = '4.0.1'
[offset 133] AUX FIELD repl-id = '44873f839ae3a57572920cdaf70399672b842691'
[offset 148] AUX FIELD repl-offset = '0'
[offset 167] \o/ RDB looks OK! \o/
[info] 1 keys read
[info] 0 expires
[info] 0 already expired

```

- redis启动读取RDB中复制信息

redis加载RDB文件，会专门处理文件中辅助字段(AUX fields）信息，把其中repl_id和repl_offset加载到实例中，分别赋给master_replid和master_repl_offset两个变量值。

- redis从实例尝试部分重新同步

redis实例重启后，从RDB文件中加载master_replid和master_repl_offset。当我们把它作为某个实例的从库时（包含如被动的cluster slave或主动执行slaveof指令)，实例向主实例上报master_replid和master_repl_offset+1；从实例同时满足以下两条件，就可以部分重新同步：

1. 从实例上报master_replid串，与主实例的master_replid1或replid2有一个相等
2. 从实例上报的master_repl_offset+1字节，还存在于主实例的复制积压缓冲区中

- redis重启时，临时调整主实例的复制积压缓冲区大小

redis的复制积压缓冲区是通过参数repl-backlog-size设置，默认1MB；为确保从实例重启后，还能部分重新同步，需设置合理的repl-backlog-size值。

1. 计算合理的repl-backlog-size值大小： 通过主库每秒增量的master复制偏移量master_repl_offset(info replication指令获取)大小，如每秒offset增加是5MB,那么主实例复制积压缓冲区要保留最近60秒写入内容，backlog_size设置就得大于300MB(60*5)。而从实例重启加载RDB文件是较耗时的过程，如重启某个重实例需120秒(RDB大小和CPU配置相关），那么主实例backlog_size就得设置至少600MB.计算公式：`backlog_size = 重启从实例时长 * 主实例offset每秒写入量`

2. 重启从实例前，调整主实例的动态调整repl-backlog-size的值： 通过config set动态调整redis的repl-backlog-size时，redis会释放当前的积压缓冲区，重新分配一个指定大小的缓冲区。 所以我们必须在从实例重启前，调整主实例的repl-backlog-size。

## 故障切换
psync2除了解决redis重启使用部分同步外，还为解决在主库故障时候从库切换为主库时候使用部分同步机制。redis从库默认开启复制积压缓冲区功能，以便从库故障切换变化master后，其他落后该从库可以从缓冲区中获取缺少的命令。该过程的实现通过两组replid、offset替换原来的master runid和offset变量实现：

第一组：master_replid和master_repl_offset：如果redis是主实例，则表示为自己的replid和复制偏移量； 如果redis是从实例，则表示为自己主实例的replid1和同步主实例的复制偏移量。

第二组：master_replid2和second_repl_offset：无论主从，都表示自己上次主实例repid1和复制偏移量；用于兄弟实例或级联复制，主库故障切换psync。初始化时, 前者是40个字符长度为0，后者是-1； 只有当主实例发生故障切换时，redis把自己replid1和master_repl_offset+1分别赋值给master_replid2和second_repl_offset。

判断是否使用部分复制条件：如果从库提供的master_replid与master的replid不同，且与master的replid2不同，或同步速度快于master； 就必须进行全量复制，否则执行部分复制。

这样发生主库故障切换，以下三种常见结构，都能进行psync:

1. 一主一从发生切换，A->B 切换变成 B->A ;
2. 一主多从发生切换，兄弟节点变成父子节点时；
3. 级别复制发生切换， A->B->C 切换变成 B->C->A

# 拓扑

## 一主一从

主库不开启持久化，从库开启AOF，这样既保证了数据安全性，同时也避免了持久化对主节点性能的干扰。 **注意**：因为主节点没有开启持久化，如果主节点重启后数据集为空，如果从节点继续复制主节点会导致从节点的数据也被清空。所以要先在从节点上执行`slaveof no one`断开主从复制。

![](/assets/images/posts/redis-slave/redis-slave-7.jpg)

## 一主多从

对于读占比较大的场景，可以把读命令发送到从节点来分担从节点压力。但对于写占比较大的场景，多个从节点会导致主就节点写命令的多次发送从而过度消耗网络带宽，同时也加重了主节点的负载影响服务稳定性

![](/assets/images/posts/redis-slave/redis-slave-8.jpg)

## 树状主从结构

从节点不但可以复制主就节点数据，同时可以作为其他从节点的主节点继续向下层复制。有效降低了主节点的复制和需要传送给从节点的数据量

![](/assets/images/posts/redis-slave/redis-slave-9.jpg)

# 应用中的问题

## 读写分离及其中的问题

在主从复制基础上实现的读写分离，可以实现 Redis 的读负载均衡。

由主节点提供写服务，由一个或多个从节点提供读服务（多个从节点既可以提高数据冗余程度，也可以最大化读负载能力）；在读负载较大的应用场景下，可以大大提高 Redis 服务器的并发量。

使用 Redis 读写分离时，需要注意下列问题。

**延迟与不一致问题**

前面已经讲到，由于主从复制的命令传播是异步的，延迟与数据的不一致不可避免。

如果应用对数据不一致的接受程度程度较低，可能的优化措施包括：

- **优化主从节点之间的网络环境（如在同机房部署）。**
- **监控主从节点延迟（通过 offset）判断，如果从节点延迟过大，通知应用不再通过该从节点读取数据。**
- **使用集群同时扩展写负载和读负载等。**

在命令传播阶段以外的其他情况下，从节点的数据不一致可能更加严重，例如连接在数据同步阶段，或从节点失去与主节点的连接时等。

从节点的 `slave-serve-stale-data` 参数便与此有关：它控制这种情况下从节点的表现；如果为 yes（默认值），则从节点仍能够响应客户端的命令，如果为 no，则从节点只能响应 info、slaveof 等少数命令。该参数的设置与应用对数据一致性的要求有关；如果对数据一致性要求很高，则应设置为 no。

**数据过期问题**

在单机版 Redis 中，存在两种删除策略：

- **惰性删除：**服务器不会主动删除数据，只有当客户端查询某个数据时，服务器判断该数据是否过期，如果过期则删除。
- **定期删除：**服务器执行定时任务删除过期数据，但是考虑到内存和 CPU 的折中（删除会释放内存，但是频繁的删除操作对 CPU 不友好），该删除的频率和执行时间都受到了限制。

在主从复制场景下，为了主从节点的数据一致性，从节点不会主动删除数据，而是由主节点控制从节点中过期数据的删除。

由于主节点的惰性删除和定期删除策略，都不能保证主节点及时对过期数据执行删除操作，因此，当客户端通过 Redis 从节点读取数据时，很容易读取到已经过期的数据。

在 Redis 3.2 中，从节点在读取数据时，增加了对数据是否过期的判断：如果该数据已过期，则不返回给客户端；将 Redis 升级到 3.2 可以解决数据过期问题。

**故障切换问题**

在没有使用哨兵的读写分离场景下，应用针对读和写分别连接不同的 Redis 节点。

当主节点或从节点出现问题而发生更改时，需要及时修改应用程序读写 Redis 数据的连接；连接的切换可以手动进行，或者自己写监控程序进行切换，但前者响应慢、容易出错，后者实现复杂，成本都不算低。

**总结：**在使用读写分离之前，可以考虑其他方法增加 Redis 的读负载能力：如尽量优化主节点（减少慢查询、减少持久化等其他情况带来的阻塞等）提高负载能力；使用 Redis 集群同时提高读负载能力和写负载能力等。

如果使用读写分离，可以使用哨兵，使主从节点的故障切换尽可能自动化，并减少对应用程序的侵入。

## 复制超时问题

主从节点复制超时是导致复制中断的最重要的原因之一

### 超时判断意义

在复制连接建立过程中及之后，主从节点都有机制判断连接是否超时，其意义在于：

1. 如果主节点判断连接超时，其会释放相应从节点的连接，从而释放各种资源，否则无效的从节点仍会占用主节点的各种资源（输出缓冲区、带宽、连接等）；此外连接超时的判断可以让主节点更准确的知道当前有效从节点的个数，有助于保证数据安全（配合前面讲到的min-slaves-to-write等参数）。
2. 如果从节点判断连接超时，则可以及时重新建立连接，避免与主节点数据长期的不一致。

### 判断机制

主从复制超时判断的核心，在于`repl-timeout`参数，该参数规定了超时时间的阈值（默认60s），对于主节点和从节点同时有效；主从节点触发超时的条件分别如下：

1. 主节点：每秒1次调用复制定时函数replicationCron()，在其中判断当前时间距离上次收到各个从节点REPLCONF ACK的时间，是否超过了repl-timeout值，如果超过了则释放相应从节点的连接。

2. 从节点：从节点对超时的判断同样是在复制定时函数中判断，基本逻辑是：如果当前处于连接建立阶段，且距离上次收到主节点的信息的时间已超过repl-timeout，则释放与主节点的连接；如果当前处于数据同步阶段，且收到主节点的RDB文件的时间超时，则停止数据同步，释放连接；如果当前处于命令传播阶段，且距离上次收到主节点的PING命令或数据的时间已超过repl-timeout值，则释放与主节点的连接。

### 导致超时的原因

1. 数据同步阶段：在主从节点进行全量复制bgsave时，主节点需要首先fork子进程将当前数据保存到RDB文件中，然后再将RDB文件通过网络传输到从节点。如果RDB文件过大，主节点在fork子进程+保存RDB文件时耗时过多，可能会导致从节点长时间收不到数据而触发超时；此时从节点会重连主节点，然后再次全量复制，再次超时，再次重连……这是个悲伤的循环。为了避免这种情况的发生，除了注意Redis单机数据量不要过大，另一方面就是适当增大repl-timeout值，具体的大小可以根据bgsave耗时来调整。
2. 命令传播阶段：如前所述，在该阶段主节点会向从节点发送PING命令，频率由repl-ping-slave-period控制；该参数应明显小于repl-timeout值(后者至少是前者的几倍)。否则，如果两个参数相等或接近，网络抖动导致个别PING命令丢失，此时恰巧主节点也没有向从节点发送数据，则从节点很容易判断超时。
3. 慢查询导致的阻塞：如果主节点或从节点执行了一些慢查询（如`keys *`或者对大数据的`hgetall`等），导致服务器阻塞；阻塞期间无法响应复制连接中对方节点的请求，可能导致复制超时。

## 复制中断问题

主从节点超时是复制中断的原因之一，除此之外，还有其他情况可能导致复制中断，其中最主要的是复制缓冲区溢出问题。

**复制缓冲区溢出**

前面曾提到过，在全量复制阶段，主节点会将执行的写命令放到复制缓冲区中，该缓冲区存放的数据包括了以下几个时间段内主节点执行的写命令：

- bgsave生成RDB文件
- RDB文件由主节点发往从节点
- 从节点清空老数据并载入RDB文件中的数据

当主节点数据量较大，或者主从节点之间网络延迟较大时，可能导致该缓冲区的大小超过了限制，此时主节点会断开与从节点之间的连接；这种情况可能引起全量复制->复制缓冲区溢出导致连接中断->重连->全量复制->复制缓冲区溢出导致连接中断……的循环。

复制缓冲区的大小由`client-output-buffer-limit slave {hard limit} {soft limit} {soft seconds}`配置，默认值为`client-output-buffer-limit slave 256MB 64MB 60`，其含义是：如果buffer大于256MB，或者连续60s大于64MB，则主节点会断开与该从节点的连接。该参数是可以通过config set命令动态配置的（即不重启Redis也可以生效）。

需要注意的是，复制缓冲区是客户端输出缓冲区的一种，主节点会为每一个从节点分别分配复制缓冲区；而复制积压缓冲区则是一个主节点只有一个，无论它有多少个从节点。
# 相关配置

```
################################# REPLICATION #################################

# slaveof <主服务器ip> <主服务器端口>
# slaveof <masterip> <masterport>

# masterauth <主服务器Redis密码>
# masterauth <master-password>

# 当slave丢失master或者同步正在进行时，如果发生对slave的服务请求
# yes则slave依然正常提供服务
# no则slave返回client错误："SYNC with master in progress"
slave-serve-stale-data yes

# 指定slave是否只读
slave-read-only yes

# 作用于全量复制阶段，控制主节点是否使用 diskless 复制（无盘复制）。
# 所谓 diskless 复制，是指在全量复制时，主节点不再先把数据写入 RDB 文件，而是直接写入 slave 的 socket 中，整个过程中不涉及硬盘。
# diskless 复制在磁盘 IO 很慢而网速很快时更有优势。
repl-diskless-sync no

# 无硬盘复制功能间隔时间，当收到第一个复制请求时，等待 5s 后再开始复制，因为要等更多 slave 一起连接过来
# 之所以设置停顿时间，是基于以下两个考虑：
#  - 向 slave 的 socket 的传输一旦开始，新连接的 slave 只能等待当前数据传输结束，才能开始新的数据传输。
#  - 多个从节点有较大的概率在短时间内建立主从复制。
repl-diskless-sync-delay 5

# 从服务器发送PING命令给主服务器的周期，repl-timeout > repl-ping-slave-period
# repl-ping-slave-period 10

# 超时时间
# repl-timeout 60
# 三种情况认为复制超时： 
# 1）slave角度，如果在repl-timeout时间内没有收到master SYNC传输的rdb snapshot数据， 
# 2）slave角度，在repl-timeout没有收到master发送的数据包或者ping。 
# 3）master角度，在repl-timeout时间没有收到REPCONF ACK确认信息。

对于内存数据量比较大的系统，可以增大repl-timeout参数。

# 是否禁用socket的NO_DELAY选项
# - 当关闭时，主节点产生的命令数据无论大小都会及时地发送给从节点，这样主从之间的延迟会变小，但增加了网络带宽的消耗，适用于主从之间的网络环境良好的场景。
# - 当开启时，主节点会合并较小的TCP数据包从而节省带宽。默认发送时间取决于linux的内核，一般默认为40毫秒。这种复杂节省了带宽但增大了主从之间的延迟。适用于主从网络环境复杂或者带宽紧张的场景
repl-disable-tcp-nodelay no

# 设置主从复制容量大小，这个backlog 是一个用来在 slaves 被断开连接时存放 slave 数据的 buffer
# repl-backlog-size 1mb
# master 不再连接 slave时backlog的存活时间。
# repl-backlog-ttl 3600
# 这是一个环形复制缓冲区，用来保存最新复制的命令。这样在slave离线的时候，不需要完全复制master的数据，如果可以执行部分同步，只需要把缓冲区的部分数据复制给slave，就能恢复正常复制状态。缓冲区的大小越大，slave离线的时间可以更长，复制缓冲区只有在有slave连接的时候才分配内存。没有slave的一段时间，内存会被释放出来。
# 业务比较繁忙（QPS高）的redis，可以适当调大此值。

# slave的优先级
slave-priority 100

# 在从服务器的数量小于3个，或者三个从服务器的延迟（lag）值都大于10S时，主服务器拒绝执行写命令
# min-slaves-to-write 3
# min-slaves-max-lag 10

#如果在复制期间，rdb复制时间超过60秒，内存缓冲区持续消耗超过64MB，或者一次性超过256MB，那么将会停止复制(失败)
# client-output-buffer-limit slave 256MB 64MB 60
# 其实对于主从复制来说，SLAVE种类的client-output-buffer存放的数据是下面三个时间内所有的master数据更新操作。
#    1）master执行rdb bgsave产生snapshot的时间 
#    2）master发送rdb到slave网络传输时间 
#    3）slave load rdb文件把数据恢复到内存的时间
# 因此业务比较繁忙（QPS高）的情况需要考虑调大该值。

# output buffer是Redis为client分配的缓冲区（这里的”client”可能是真正的client，也可能是slave或monitor），若为某个客户端分配的output buffer超过了预留大小，Redis可能会根据配置策略关闭与该端的连接。
# `client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>`
# class ： 客户端种类，normal（普通的客户端）、slave（从库的复制客户端和MONITOR）、pubsub（发布与订阅的客户端）。 
# hard limit ：若output buffer大小超过该值，Redis会立即关闭与对应client的连接 
# soft limit ：若output buffer大小超过soft limit且这种情况的持续时间超过soft seconds，则Redis会关闭与对应client的连接。 
# 默认的配置如下： 
# client-output-buffer-limit normal 0 0 0 
# client-output-buffer-limit slave 256mb 64mb 60 
# client-output-buffer-limit pubsub 32mb 8mb 60
```
# 参考资料

https://mp.weixin.qq.com/s/NlSby0pX4Ddh7Kz_-wunnQ

https://zhuanlan.zhihu.com/p/47719810

https://www.jianshu.com/p/54dabc470eb6

https://www.d1blog.com/shujuku/25.html

https://www.jianshu.com/p/6fe7c56e487c

《Redis开发与运维》

https://mp.weixin.qq.com/s/mO-NTdRkawxAPXggXXqclw