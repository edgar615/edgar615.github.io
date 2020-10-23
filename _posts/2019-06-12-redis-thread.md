---
layout: post
title: redis线程模型
date: 2019-06-12
categories:
    - redis
comments: true
permalink: redis-thread.html
---

redis使用的单线程模型

1. redis 会将每个客户端都关联一个指令队列。客户端的指令通过队列来按顺序处理，先到先服务。
2. 在一个客户端的指令队列中的指令是顺序执行的，但是多个指令队列中的指令是无法保证顺序的
3. redis 同样也会为每个客户端关联一个响应队列，通过响应队列来顺序地将指令的返回结果回复给客户端。
4. 同样，一个响应队列中的消息可以顺序的回复给客户端，多个响应队列之间是无法保证顺序的。
5. 所有的客户端的队列中的指令或者响应，redis 每次都只能处理一个，同一时间绝对不会处理超过一个指令或者响应。

# 为什么redis能够快速执行

- redis 将所有数据放在内存中，绝大部分请求是纯粹的内存操作，内存的响应时长大约为 100 纳秒，这是 redis 的 QPS 过万的重要基础。
- 采用单线程,避免了不必要的上下文切换和竞争条件
- 非阻塞I/O，Redis采用epoll做为I/O多路复用技术的实现，再加上Redis自身的事件处理模型将epoll中的连接，读写，关闭都转换为了事件，不在I/O上浪费过多的时间。
- Redis采用单线程模型，每条命令执行如果占用大量时间，会造成其他线程阻塞，对于Redis这种高性能服务是致命的，所以Redis是面向高速执行的数据库。
- Redis为了高效的处理客户端的事件，并没有将持久化文件放在主线程里面进行处理，而是Redis在适当的时机fork子进程来异步的处理这种任务，Redis会fork子进程进行处理持久化文件操作（将数据写到RDB 文件中）。Redis还有一组异步任务处理线程，用于处理不需要主线程同步处理的工作，即处理一些低级别的事件（AOF文件重写）。


# IO多路复用
I/O multiplexing 这里面的 multiplexing 指的其实是在单个线程通过记录跟踪每一个Sock(I/O流)的状态来同时管理多个I/O流. 发明它的原因，是尽量多的提高服务器的吞吐能力。

参考下图(来源于参考资料)

![](/assets/images/posts/redis-thread/redis-thread-1.png)

在上图中，redis 需要处理 3 个 IO 请求，同时把 3 个请求的结果返回给客户端，所以总共需要处理 6 个 IO 事件，由于 redis 是单线程模型，同一时间只能处理一个 IO 事件，于是 redis 需要在合适的时间暂停对某个 IO 事件的处理，转而去处理另一个 IO 事件，这样 redis 就好比一个开关，当开关拨到哪个 IO 事件这个电路上，就处理哪个 IO 事件，其他 IO 事件就暂停处理了。这就是IO多路复用技术。




# 参考资料

https://cloud.tencent.com/developer/article/1403767

https://www.zhihu.com/question/32163005

https://www.liangzl.com/get-article-detail-26853.html
