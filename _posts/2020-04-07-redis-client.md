---
layout: post
title: redis客户端通信协议
date: 2020-04-07
categories:
    - redis
comments: true
permalink: redis-client.html
---

# 客户端通信协议

http://redisdoc.com/topic/protocol.html

客户端和服务器通过 TCP 连接来进行数据交互， 服务器默认的端口号为 6379 。
客户端和服务器发送的命令或数据一律以 \r\n （CRLF）结尾。

## 发送命令
格式

    *<参数数量> CRLF
    $<参数 1 的字节数量> CRLF
    <参数 1 的数据> CRLF
    ...
    $<参数 N 的字节数量> CRLF
    <参数 N 的数据> CRLF

示例

    *3
    $3
    SET
    $5
    mykey
    $7
    myvalue

上述格式是格式化显示的结果，实际传输的格式如下

	"*3\r\n$3\r\nSET\r\n$5\r\nmykey\r\n$7\r\nmyvalue\r\n"

## 回复命令
redis的返回结果类型分为五种

- 状态回复（status reply）的第一个字节是 "+"
- 错误回复（error reply）的第一个字节是 "-"
- 整数回复（integer reply）的第一个字节是 ":"
- 字符串回复（bulk reply）的第一个字节是 "$"
- 多条字符串回复（multi bulk reply）的第一个字节是 "*"

### 状态回复
一个状态回复（或者单行回复，single line reply）是一段以 "+" 开始、 "\r\n" 结尾的单行字符串。
示例：

	+OK

客户端库应该返回 "+" 号之后的所有内容。 比如在在上面的这个例子中， 客户端就应该返回字符串 "OK" 。
状态回复通常由那些不需要返回数据的命令返回，这种回复不是二进制安全的，它也不能包含新行。状态回复的额外开销非常少，只需要三个字节（开头的 "+" 和结尾的 CRLF）。

### 错误回复
错误回复和状态回复非常相似， 它们之间的唯一区别是， 错误回复的第一个字节是 "-" ， 而状态回复的第一个字节是 "+" 。
错误回复只在某些地方出现问题时发送： 比如说， 当用户对不正确的数据类型执行命令， 或者执行一个不存在的命令， 等等。一个客户端库应该在收到错误回复时产生一个异常。

示例：

	-ERR unknown command 'foobar'
	-WRONGTYPE Operation against a key holding the wrong kind of value

在 "-" 之后，直到遇到第一个空格或新行为止，这中间的内容表示所返回错误的类型。

**ERR** 是一个通用错误，而 **WRONGTYPE** 则是一个更特定的错误。 一个客户端实现可以为不同类型的错误产生不同类型的异常， 或者提供一种通用的方式， 让调用者可以通过提供字符串形式的错误名来捕捉（trap）不同的错误。

### 整数回复
整数回复就是一个以 ":" 开头， CRLF 结尾的字符串表示的整数。比如说， ":0\r\n" 和 ":1000\r\n" 都是整数回复。
返回整数回复的其中两个命令是 INCR 和 LASTSAVE 。 被返回的整数没有什么特殊的含义， INCR 返回键的一个自增后的整数值， 而 LASTSAVE 则返回一个 UNIX 时间戳， 返回值的唯一限制是这些数必须能够用 64 位有符号整数表示。
整数回复也被广泛地用于表示逻辑真和逻辑假： 比如 EXISTS 和 SISMEMBER 都用返回值 1 表示真， 0 表示假。
其他一些命令， 比如 SADD 、 SREM 和 SETNX ， 只在操作真正被执行了的时候， 才返回 1 ， 否则返回 0 。
以下命令都返回整数回复： SETNX 、 DEL 、 EXISTS 、 INCR 、 INCRBY 、 DECR 、 DECRBY 、 DBSIZE 、 LASTSAVE 、 RENAMENX 、 MOVE 、 LLEN 、 SADD 、 SREM 、 SISMEMBER 、 SCARD 。

### 字符串回复/批量回复
服务器使用批量回复来返回二进制安全的字符串，字符串的最大长度为 512 MB 。服务器发送的内容中：

    第一字节为 "$" 符号
    接下来跟着的是表示实际回复长度的数字值
    之后跟着一个 CRLF
    再后面跟着的是实际回复数据
    最末尾是另一个 CRLF

例如

	客户端：GET mykey
	服务器：foobar
	
对于这个的 GET 命令，服务器实际发送的内容为：

	"$6\r\nfoobar\r\n"

如果被请求的值不存在， 那么批量回复会将特殊值 -1 用作回复的长度值， 就像这样：

	客户端：GET non-existing-key
	服务器：$-1

这种回复称为空批量回复（NULL Bulk Reply）。
当请求对象不存在时，客户端应该返回空对象，而不是空字符串。

### 多条字符串回复/多条批量回复
像 LRANGE 这样的命令需要返回多个值， 这一目标可以通过多条批量回复来完成。
多条批量回复是由多个回复组成的数组， 数组中的每个元素都可以是任意类型的回复， 包括多条批量回复本身。
多条批量回复的第一个字节为 "*" ， 后跟一个字符串表示的整数值， 这个值记录了多条批量回复所包含的回复数量， 再后面是一个 CRLF 。

	客户端： LRANGE mylist 0 3
	服务器： *4
	服务器： $3
	服务器： foo
	服务器： $3
	服务器： bar
	服务器： $5
	服务器： Hello
	服务器： $5
	服务器： World

在上面的示例中，服务器发送的所有字符串都由 CRLF 结尾。
正如你所见到的那样， 多条批量回复所使用的格式， 和客户端发送命令时使用的统一请求协议的格式一模一样。 它们之间的唯一区别是：

    统一请求协议只发送批量回复。
    而服务器应答命令时所发送的多条批量回复，则可以包含任意类型的回复。

以下例子展示了一个多条批量回复， 回复中包含四个整数值， 以及一个二进制安全字符串：

	*5\r\n
	:1\r\n
	:2\r\n
	:3\r\n
	:4\r\n
	$6\r\n
	foobar\r\n

在回复的第一行， 服务器发送 *5\r\n ， 表示这个多条批量回复包含 5 条回复， 再后面跟着的则是 5 条回复的正文。

多条批量回复也可以是空白的（empty）， 就像这样：

客户端： LRANGE nokey 0 1
服务器： *0\r\n

无内容的多条批量回复（null multi bulk reply）也是存在的， 比如当 BLPOP 命令的阻塞时间超过最大时限时， 它就返回一个无内容的多条批量回复， 这个回复的计数值为 -1 ：

客户端： BLPOP key 1
服务器： *-1\r\n

客户端库应该区别对待空白多条回复和无内容多条回复： 当 Redis 返回一个无内容多条回复时， 客户端库应该返回一个 null 对象， 而不是一个空数组。

### 多条批量回复中的空元素
多条批量回复中的元素可以将自身的长度设置为 -1 ， 从而表示该元素不存在， 并且也不是一个空白字符串（empty string）。
当 SORT 命令使用 GET pattern 选项对一个不存在的键进行操作时， 就会发生多条批量回复中带有空白元素的情况。
以下例子展示了一个包含空元素的多重批量回复：

	服务器： *3
	服务器： $3
	服务器： foo
	服务器： $-1
	服务器： $3
	服务器： bar

其中， 回复中的第二个元素为空。对于这个回复， 客户端库应该返回类似于这样的回复：

	["foo", nil, "bar"]

# client命令
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
