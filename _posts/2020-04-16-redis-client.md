---
layout: post
title: redis Client命令
date: 2020-04-15
categories:
    - redis
comments: true
permalink: redis-client.html
---

**CLIENT LIST** 返回所有连接到服务器的客户端信息和统计数据

    127.0.0.1:6379> client list
    id=10227 addr=27.38.49.148:32595 fd=105 name= age=26102951 idle=26102951 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=info
    id=10230 addr=27.38.49.148:32025 fd=117 name= age=26102920 idle=26102920 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=info
    id=10229 addr=27.38.49.148:31852 fd=116 name= age=26102951 idle=26102951 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=info
    id=10245 addr=27.38.49.148:32080 fd=124 name= age=26102768 idle=26102768 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=info

下面是各字段的含义：:

- id: 唯一的64位的客户端ID(Redis 2.8.12加入)。
- addr: 客户端的地址和端口
- fd: 套接字所使用的文件描述符
- age: 以秒计算的已连接时长
- idle: 以秒计算的空闲时长
- flags: 客户端 flag，后面描述
- db: 该客户端正在使用的数据库 ID
- sub: 已订阅频道的数量
- psub: 已订阅模式的数量
- multi: 在事务中被执行的命令数量
- qbuf: 输入缓冲区的长度（字节为单位， 0 表示没有分配输入缓冲区）
- qbuf-free: 输入缓冲区剩余空间的长度（字节为单位， 0 表示没有剩余空间）
- obl: 输出缓冲区（固定缓冲区）的长度（字节为单位， 0 表示没有分配输出缓冲区）
- oll: 输出缓冲区列表（动态缓冲区列表）包含的对象数量（当输出缓冲区没有剩余空间时，命令回复会以字符串对象的形式被入队到这个队列里）
- omem: 输出缓冲区和输出列表占用的内存总量
- events: 文件描述符事件
- cmd: 最近一次执行的命令

**输入缓存区**
redis会为每个客户端分配输入缓冲区，它的作用是将客户端发送的命令临时保存，同时redis会从输入缓冲区拉取命令并执行。

- 一旦某个客户端的输入缓冲区超过1G，客户端将会被关闭
- 输入缓冲区不受maxmemory控制，如果输入缓冲区+实际存储数据超过了maxmemory，可能会产生数据丢失、键值淘汰、OOM等情况

输入缓冲区过大主要是因为redis的处理速度跟不上输入缓冲区的输入速度，且每次进入输入缓冲区的命令包含了大量的bigkey。还有一种情况是redis发送了阻塞，短期内不能处理命令，造成了客户端输入的命令积压在输入缓冲区
除了**client list**外还可以通过 **info clients**找到最大是输入缓冲区

    127.0.0.1:6379> info clients
    # Clients
    connected_clients:237
    client_longest_output_list:0
    client_biggest_input_buf:0 //最大的输入缓冲区
    blocked_clients:0

**输出缓冲区**
redis为每个客户端分配了输出缓冲区，用于保存命令执行的结果返回给客户端，为redis与客户端的交互返回结果提供缓冲。根据客户端的不同，输入缓冲区分为三种：

- normal 普通客户端
- slave slave客户端
- pubsub 发布订阅客户端

可以通过client-output-buffer-limit参数来设置输出缓冲区

	client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>

- class 客户端类型 normal slave pubsub
- hard limit 如果客户端使用的输出缓冲区打印hard limit，客户端会被立即关闭
- soft limit和 soft seconds 如果客户端使用的输出缓冲区超过了soft limit并且持续了soft seconds秒，客户端会被立即关闭

redis的默认设置是

    client-output-buffer-limit normal 0 0 0
    client-output-buffer-limit slave 256mb 64mb 60
    client-output-buffer-limit pubsub 32mb 8mb 60

输出缓冲区也不受maxmemory控制，如果输出缓冲区+实际存储数据超过了maxmemory，可能会产生数据丢失、键值淘汰、OOM等情况。
输出缓冲区由两部分组成：固定缓冲区(16KB)和动态缓冲区，其中固定缓冲区返回比较小的执行结果，动态缓冲区返回毕竟大的结果。
固定缓冲区使用的是字节数组，动态缓冲区使用的列表。当固定缓冲区存满后会将redis新的返回结果存放在动态缓冲区的队列中，队列中的每个对象就是每个返回结果。

除了**client list**外还可以通过 **info clients**找到最大是输入缓冲区

    127.0.0.1:6379> info clients
    # Clients
    connected_clients:237
    client_longest_output_list:0//最大的输出缓冲区
    client_biggest_input_buf:0 
    blocked_clients:0

相比输入缓冲区，输出缓冲区出现异常的概率更大，预防方法如下：

- 按上面的方法监控
- 限制普通客户端饿输出缓冲区
- 适当增大slave的输出缓冲区，如果master节点写入较大，slave客户端的输出缓冲区可能会比较大，一旦slave的客户端被kill，会造成重连
- 限制容易让输出缓冲区增大的命令，例如高并发下的monitor命令
- 及时监控内存的抖动

**限制客户端连接数**
maxclients参数，默认值10000

    127.0.0.1:6379> info clients
    # Clients
    connected_clients:237 //连接数
    client_longest_output_list:0
    client_biggest_input_buf:0 
    blocked_clients:0

**设置连接的最大空闲时间**
timeout参数 默认值0
一旦客户端连接的idel超过了timeout，连接将会自动关闭

**检测TCP连接活性**
tcp-keepalive参数，为0不检测


**客户端flag**
客户端 flag 可以由以下部分组成：

- O: 客户端是 MONITOR 模式下的附属节点（slave）
- S: 客户端是一般模式下（normal）的附属节点
- M: 客户端是主节点（master）
- x: 客户端正在执行事务
- b: 客户端正在等待阻塞事件
- i: 客户端正在等待 VM I/O 操作（已废弃）
- d: 一个受监视（watched）的键已被修改， EXEC 命令将失败
- c: 在将回复完整地写出之后，关闭链接
- u: 客户端未被阻塞（unblocked）
- U: 通过Unix套接字连接的客户端
- r: 客户端是只读模式的集群节点
- A: 尽可能快地关闭连接
- N: 未设置任何 flag
