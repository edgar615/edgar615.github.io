---
layout: post
title: Linux IO（3）- I/O多路复用
date: 2019-09-17
categories:
    - linux
comments: true
permalink: linux-io-multiplexing.html
---

# 1. Socket 编程模型

在 UNIX 系的操作系统中，一个 Socket 文件内部类似一个双向的管道。因此，非常适用于进程间通信。在网络当中，本质上并没有发生变化。网络中的 Socket 一端连接 Buffer， 一端连接应用——也就是进程。网卡的数据会进入 Buffer，Buffer 经过协议栈的处理形成 Socket 结构。通过这样的设计，进程读取 Socket 文件，可以从 Buffer 中对应节点读走数据。

![](/assets/images/posts/linux-io/linux-io-51.png)

如上图所示，Socket 连接了应用和协议，如果应用层的程序想要传输数据，就创建一个 Socket。应用向 Socket 中写入数据，相当于将数据发送给了另一个应用。应用从 Socket 中读取数据，相当于接收另一个应用发送的数据。而具体的操作就是由 Socket 进行封装。具体来说，对于 UNIX 系的操作系统，是利用 Socket 文件系统，**Socket 是一种特殊的文件——每个都是一个双向的管道**。一端是应用，一端是缓冲区。

那么作为一个服务端的应用，如何知道有哪些 Socket 呢？也就是，哪些客户端连接过来了呢？这是就需要一种特殊类型的 Socket，也就是服务端 Socket 文件。

![](/assets/images/posts/linux-io/linux-io-52.png)

如上图所示，当有客户端连接服务端时，服务端 Socket 文件中会写入这个客户端 Socket 的文件描述符。进程可以通过 accept() 方法，从服务端 Socket 文件中读出客户端的 Socket 文件描述符，从而拿到客户端的 Socket 文件。

程序员实现一个网络服务器的时候，会先手动去创建一个服务端 Socket 文件。服务端的 Socket 文件依然会存在操作系统内核之中，并且会绑定到某个 IP 地址和端口上。以后凡是发送到这台机器、目标 IP 地址和端口号的连接请求，在形成了客户端 Socket 文件之后，文件的文件描述符都会被写入到服务端的 Socket 文件中。应用只要调用 accept 方法，就可以拿到这些客户端的 Socket 文件描述符，这样服务端的应用就可以方便地知道有哪些客户端连接了进来。

而每个客户端对这个应用而言，都是一个文件描述符。如果需要读取某个客户端的数据，就读取这个客户端对应的 Socket 文件。如果要向某个特定的客户端发送数据，就写入这个客户端的 Socket 文件。

# 2. I/O多路复用

应用程序通常需要处理来自多条事件流中的事件，比如我现在用的电脑，需要同时处理键盘鼠标的输入、中断信号等等事件，再比如web服务器如nginx，需要同时处理来来自N个客户端的事件。

>  逻辑控制流在时间上的重叠叫做 **并发**

而CPU单核在同一时刻只能做一件事情，一种解决办法是对CPU进行时分复用(多个事件流将CPU切割成多个时间片，不同事件流的时间片交替进行)。在计算机系统中，我们用线程或者进程来表示一条执行流，通过不同的线程或进程在操作系统内部的调度，来做到对CPU处理的时分复用。这样多个事件流就可以并发进行，不需要一个等待另一个太久，在用户看起来他们似乎就是并行在做一样。

但凡事都是有成本的。线程/进程也一样，有这么几个方面：

1. 线程/进程创建成本
2. CPU切换不同线程/进程成本
3. 多线程的资源竞争

有没有一种可以在单线程/进程中处理多个事件流的方法呢？一种答案就是**IO多路复用**。

因此IO多路复用解决的本质问题是在**用更少的资源完成更多的事**。

**I/O multiplexing 这里面的 multiplexing 指的其实是在单个线程通过记录跟踪每一个Sock(I/O流)的状态来同时管理多个I/O流**. 发明它的原因，是尽量多的提高服务器的吞吐能力。

在上面的讨论当中，进程拿到了它关注的所有 Socket，也称作关注的集合（Intersting Set）。如下图所示，这种过程相当于进程从所有的 Socket 中，筛选出了自己关注的一个子集，但是这时还有一个问题没有解决：进程如何监听关注集合的状态变化，比如说在有数据进来，如何通知到这个进程？

![](/assets/images/posts/linux-io/linux-io-53.png)

其实更准确地说，一个线程需要处理所有关注的 Socket 产生的变化，或者说消息。实际上一个线程要处理很多个文件的 I/O。所有关注的 Socket 状态发生了变化，都由一个线程去处理，构成了 I/O 的多路复用问题。如下图所示：

![](/assets/images/posts/linux-io/linux-io-54.png)

**操作系统如何同时监视多个socket的数据？**

一个 Socket 文件，可以由多个进程使用；而一个进程，也可以使用多个 Socket 文件。进程和 Socket 之间是多对多的关系。另一方面，一个 Socket 也会有不同的事件类型。因此操作系统很难判断，将哪样的事件给哪个进程。

这样在进程内部就需要一个数据结构来描述自己会关注哪些 Socket 文件的哪些事件（读、写、异常等）。通常有两种考虑方向，一种是利用线性结构，比如说数组、链表等，这类结构的查询需要遍历。每次内核产生一种消息，就遍历这个线性结构。看看这个消息是不是进程关注的？另一种是索引结构，内核发生了消息可以通过索引结构马上知道这个消息进程关不关注。

>  假如能够预先传入一个socket列表，**如果列表中的socket都没有数据，挂起进程，直到有一个socket收到数据，唤醒进程**。这种方法很直接，也是**select**的设计思想。

在系统底层，IO 多路复用有 3 种实现机制：

- select
- poll
- epoll（只有Linux支持，比如BSD上面对应的实现是kqueue）

**注意：select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间**

# 3. select

select的实现思路很直接。select 允许用户传入 3 个集合：read_fd_set中放入的是当数据可以读取时进程关心的 Socket；write_fd_set是当数据可以写入时进程关心的 Socket；error_fd_set是当发生异常时进程关心的 Socket。每次 select 操作会阻塞当前线程，在阻塞期间所有操作系统产生的每个消息，都会通过遍历的手段查看是否在 3 个集合当中。

假如程序同时监视如下图的sock1、sock2和sock3三个socket，那么在调用select之后，操作系统把进程A分别加入这三个socket的等待队列中。

![](/assets/images/posts/linux-io/linux-io-29.jpg)

当任何一个socket收到数据后，中断程序将唤起进程。下图展示了sock2接收到了数据的处理流程。

![](/assets/images/posts/linux-io/linux-io-30.jpg)

所谓唤起进程，就是将进程从所有的等待队列中移除，加入到工作队列里面。如下图所示。

![](/assets/images/posts/linux-io/linux-io-31.jpg)

经由这些步骤，当进程A被唤醒后，它知道至少有一个socket接收了数据。**程序只需遍历一遍socket列表**，就可以得到就绪的socket。

这种简单方式**行之有效**，在几乎所有操作系统都有对应的实现。

**但是简单的方法往往有缺点，主要是：**

- 每次调用select都需要将进程加入到所有监视socket的等待队列，每次唤醒都需要从每个队列中移除。这里涉及了两次遍历，而且每次都要将整个fds列表传递给内核，有一定的开销。正是因为遍历操作开销大，出于效率的考量，才会规定select的最大监视数量，默认只能监视**1024个socket**。
- 进程被唤醒后，程序并不知道哪些socket收到数据，还需要遍历一次。

# 4. poll

从写程序的角度来看，select 并不是一个很好的编程模型。一个好的编程模型应该直达本质，当网络请求发生状态变化的时候，核心是会发生事件。一个好的编程模型应该是直接抽象成消息：**用户不需要用 select 来设置自己的集合，而是可以通过系统的 API 直接拿到对应的消息，从而处理对应的文件描述符**。

- poll 是一个阻塞调用，它将某段时间内操作系统内发生的且进程关注的消息告知用户程序；

- 用户程序通过直接调用 poll 函数拿到消息；

- poll 函数的第一个参数告知内核 poll 关注哪些 Socket 及消息类型；

- poll 调用后，经过一段时间的等待（阻塞），就拿到了是一个消息的数组；

- 通过遍历这个数组中的消息，能够知道关联的文件描述符和消息的类型；

- 通过消息类型判断接下来该进行读取还是写入操作；

- 通过文件描述符，可以进行实际地读、写、错误处理。

poll 虽然优化了编程模型，但是从性能角度分析，它和 select 差距不大。因为内核在产生一个消息之后，依然需要遍历 poll 关注的所有文件描述符来确定这条消息是否跟用户程序相关。

# 5. epoll

为了解决上述问题，epoll 通过更好的方案实现了从操作系统订阅消息。epoll 将进程关注的文件描述符存入一棵二叉搜索树，通常是红黑树的实现。在这棵红黑树当中，Key 是 Socket 的编号，值是这个 Socket 关注的消息。因此，当内核发生了一个事件：比如 Socket 编号 1000 可以读取。这个时候，可以马上从红黑树中找到进程是否关注这个事件。

另外当有关注的事件发生时，epoll 会先放到一个队列当中。当用户调用epoll_wait时候，就会从队列中返回一个消息。epoll 函数本身是一个构造函数，只用来创建红黑树和队列结构。epoll_wait调用后，如果队列中没有消息，也可以马上返回。因此epoll是一个非阻塞模型。

对于每一个新的客户端连接，都使用 accept 拿到这个连接的文件描述符，并且创建一个客户端的 Socket。然后通过epoll_ctl将客户端的文件描述符和关注的消息类型放入 epoll 的红黑树。操作系统每次监测到一个新的消息产生，就会通过红黑树对比这个消息是不是进程关注的

epoll通过以下一些措施来改进效率。

**措施一：功能分离**

select低效的原因之一是将“维护等待队列”和“阻塞进程”两个步骤合二为一。如下图所示，每次调用select都需要这两步操作，然而大多数应用场景中，需要监视的socket相对固定，并不需要每次都修改。epoll将这两个操作分开，先用epoll_ctl维护等待队列，再调用epoll_wait阻塞进程。显而易见的，效率就能得到提升。

![](/assets/images/posts/linux-io/linux-io-32.jpg)

**措施二：就绪列表**

select低效的另一个原因在于程序不知道哪些socket收到数据，只能一个个遍历。如果内核维护一个“就绪列表”，引用收到数据的socket，就能避免遍历。如下图所示，计算机共有三个socket，收到数据的sock2和sock3被rdlist（就绪列表）所引用。当进程被唤醒后，只要获取rdlist的内容，就能够知道哪些socket收到数据。

![](/assets/images/posts/linux-io/linux-io-33.jpg)

# 4. 三种实现的区别

**用户态将文件描述符传入内核的方式**

- select：创建3个文件描述符集并拷贝到内核中,分别监听读、写、异常动作。**这里受到单个进程可以打开的fd数量限制,默认是1024**
- poll：将传入的struct pollfd结构体数组拷贝到内核中进行监听
- epoll：执行epoll_create会在内核的高速cache区中建立一颗红黑树以及就绪链表(该链表存储已经就绪的文件描述符)。接着用户执行的epoll_ctl函数添加文件描述符会在红黑树上增加相应的结点

**内核态检测文件描述符是否可读可写的方式**

- select：**采用轮询方式**,遍历所有fd,最后返回一个描述符读写操作是否就绪的mask掩码,根据这个掩码给fd_set赋值。 
- poll：**同样采用轮询方式**,查询每个fd的状态,如果就绪则在等待队列中加入一项并继续遍历。 
- epoll：**采用回调机制**。在执行epoll_ctl的add操作时,不仅将文件描述符放到红黑树上,而且也注册了回调函数,内核在检测到某文件描述符可读/可写时会调用回调函数,该回调函数将文件描述符放在就绪链表中。

**如何找到就绪的文件描述符并传递给用户态**

- select：将之前传入的fd_set拷贝传出到用户态并返回就绪的文件描述符总数。用户态并不知道是哪些文件描述符处于就绪态,需要遍历来判断。 
- poll：将之前传入的fd数组拷贝传出用户态并返回就绪的文件描述符总数。用户态并不知道是哪些文件描述符处于就绪态,需要遍历来判断。 
- epoll：epoll_wait只用观察就绪链表中有无数据即可,最后将链表的数据返回给数组并返回就绪的数量。内核将就绪的文件描述符放在传入的数组中,所以只用遍历依次处理即可。这里返回的文件描述符是通过mmap让内核和用户空间共享同一块内存实现传递的,减少了不必要的拷贝。

**继续重新监听时如何重复以上步骤**

- select：将新的监听文件描述符集合拷贝传入内核中,继续以上步骤。 
- poll：将新的struct pollfd结构体数组拷贝传入内核中,继续以上步骤。 
- epoll：无需重新构建红黑树,直接沿用已存在的即可。

通过以上步骤我们可以发现以下几点：

- select和poll的动作基本一致,只是poll采用链表来进行文件描述符的存储,而select采用fd标注位来存放,所以select会受到最大连接数的限制,而poll不会。
- select、poll、epoll虽然都会返回就绪的文件描述符数量。但是select和poll并不会明确指出是哪些文件描述符就绪,而epoll会。造成的区别就是,系统调用返回后,调用select和poll的程序需要遍历监听的整个文件描述符找到是谁处于就绪,而epoll则直接处理就行了。
- select、poll都需要将有关文件描述符的数据结构拷贝进内核,最后再拷贝出来。而epoll创建的有关文件描述符的数据结构本身就存于内核态中,系统调用返回时也采用mmap共享存储区,需要拷贝的次数大大减少。
- select、poll采用轮询的方式来检查文件描述符是否处于就绪态,而epoll采用回调机制。造成的结果就是,随着fd的增加,select和poll的效率会线性降低,而epoll不会受到太大影响,除非活跃的socket很多。
- 最后总结一下,epoll比select和poll高效的原因主要有两点： 

1. 减少了用户态和内核态之间的文件描述符拷贝 
2. 减少了对就绪文件描述符的遍历

适应场景

- select、poll适用于所监视的文件描述符数量较少的场景
- 当活动连接比较多的时候，epoll_wait的效率未必比select和poll高，因为此时回调函数被触发的过于频繁，因此epoll_wait适用于连接数量多，但活动连接较少的情况

# 5. 参考资料

https://www.liangzl.com/get-article-detail-26853.html

https://zhuanlan.zhihu.com/p/64138532