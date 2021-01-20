---
layout: post
title: Linux用户空间和内核空间
date: 2019-10-30
categories:
    - linux
comments: true
permalink: linux-user-kernel-space.html
---

以前在学习kafka的时候了解一个零拷贝的知识，里面说了用户空间、内核空间。一直没去看到底是怎么。这次抽时间学习了一下（基本抄的）

# 1. 用户空间和内核空间

Linux 按照特权等级，把进程的运行空间分为内核空间和用户空间，分别对应着下图中， CPU 特权等级的 Ring 0 和 Ring 3。内核空间（Ring 0）具有最高权限，可以直接访问所有资源；用户空间（Ring 3）只能访问受限资源，不能直接访问内存等硬件设备，必须通过系统调用陷入到内核中，才能访问这些特权资源。

![](/assets/images/posts/linux-user-kernel-space/linux-user-kernel-space-10.png)

- User space：用户空间，用户程序的运行空间
- Kernel space：内核空间， Linux 内核的运行空间

不同的空间，拥有自己的内存地址范围，在32位操作系统中，一般将最高的1G字节划分为内核空间，供内核使用，而将较低的3G字节划分为用户空间，供各个进程使用。

![](/assets/images/posts/linux-user-kernel-space/linux-user-kernel-space-1.jpg)

其中：

- 内核空间中存放的是内核代码和数据，而进程的用户空间中存放的是用户程序的代码和数据。
- 进程在运行的时候，在内核空间和用户空间各有一个堆栈。
- 用户空间中，每个进程的用户空间是互相独立的，互不相干。
- 内核空间中，绝大部分是共享的，并不是完全共享，因为内核空间中，不同进程的内核栈之间是不共享的。
- 内核空间总是驻留在内存中，它是为操作系统的内核保留的。应用程序是不允许直接在该区域进行读写或直接调用内核代码定义的函数的。
- 用户空间不能直接调用系统资源，内核空间和用户空间一般通过系统调用进行通信。 

# 2. 内核态和用户态

用户空间中的代码被限制了只能使用一个局部的内存空间，我们说这些程序在用户态（User Mode） 执行。内核空间中的代码可以访问所有内存，我们称这些程序在内核态（Kernal Mode） 执行。

当一个进程执行系统调用而陷入内核代码中执行时，称进行处于内核运行态（内核态）。此时处理器处于特权级别最高的（0级）内核代码中执行。当进程处于内核态时，执行的内核代码会使用当前进程的内核栈。每个进程都有自己的内核栈。

当进行在执行用户自己的代码时，则称其处于用户运行态（用户态）。此时处理器在特权级最低的（3级）用户代码中运行。当正在执行用户程序而突然被中断程序中断时，此时用户程序也可象征性的称为处于进行的内核态，因为中断处理程序使用当前进程的内核栈。

**内核态可以执行任意命令，调用系统的一切资源，而用户态只能执行简单的运算，不能直接调用系统资源。用户态必须通过系统接口（System Call），才能向内核发出指令。**

内核程序执行在内核态（Kernal Mode），用户程序执行在用户态（User Mode）。当发生系统调用时，用户态的程序发起系统调用。因为系统调用中牵扯特权指令，用户态程序权限不足，因此会中断执行，也就是 Trap（Trap 是一种中断）。

发生中断后，当前 CPU 执行的程序会中断，跳转到中断处理程序。内核程序开始执行，也就是开始处理系统调用。内核处理完成后，主动触发 Trap，这样会再次发生中断，切换回用户态工作。

![](/assets/images/posts/linux-user-kernel-space/linux-user-kernel-space-5.png)

比如，当用户进程启动一个 bash 时，它会通过 getpid() 对内核的 pid 服务发起系统调用，获取当前用户进程的 ID。

![](/assets/images/posts/linux-user-kernel-space/linux-user-kernel-space-2.png)

当用户进程通过 cat 命令查看主机配置时，它会对内核的文件子系统发起系统调用：

- 内核空间可以访问所有的 CPU 指令和所有的内存空间、I/O 空间和硬件设备
- 用户空间只能访问受限的资源，如果需要特殊权限，可以通过系统调用获取相应的资源。
- 用户空间允许页面中断，而内核空间则不允许
- 内核空间和用户空间是针对线性地址空间的
- x86 CPU 中用户空间是 0-3G 的地址范围，内核空间是 3G-4G 的地址范围
- x86_64 CPU 用户空间地址范围为0x0000000000000000–0x00007fffffffffff，内核地址空间为 0xffff880000000000-最大地址
- 所有内核进程（线程）共用一个地址空间，而用户进程都有各自的地址空间

有了用户空间和内核空间的划分后，Linux 内部层级结构可以分为三部分，从最底层到最上层依次是硬件、内核空间和用户空间，如下图所示：

![](/assets/images/posts/linux-user-kernel-space/linux-user-kernel-space-3.png)

例如服务端读取一个文件内容，然后发送给用户，大致的过程日下：

![](/assets/images/posts/linux-user-kernel-space/linux-user-kernel-space-4.png)

数据从I/O设备中读取之后，会先存放在内核buf中，系统调用完成之后，会将数据从内核buf拷贝到用户空间的buf，这一段就是零拷贝需要处理的逻辑

# 3. 内核态线程和用户态线程

## 3.1. 进程和线程

一个应用程序启动后会在内存中创建一个执行副本，这就是进程。Linux 的内核是一个 Monolithic Kernel（宏内核），因此可以看作一个进程。也就是开机的时候，磁盘的内核镜像被导入内存作为一个执行副本，成为内核进程。

进程可以分成用户态进程和内核态进程两类。用户态进程通常是应用程序的副本，内核态进程就是内核本身的进程。如果用户态进程需要申请资源，比如内存，可以通过系统调用向内核申请。

**那么用户态进程如果要执行程序，是否也要向内核申请呢？**

程序在现代操作系统中并不是以进程为单位在执行，而是以一种轻量级进程（Light Weighted Process），也称作线程（Thread）的形式执行。

一个进程可以拥有多个线程。进程创建的时候，一般会有一个主线程随着进程创建而创建。

如果进程想要创造更多的线程，就需要思考一件事情，这个线程创建在用户态还是内核态。**进程可以通过 API 创建用户态的线程，也可以通过系统调用创建内核态的线程**

## 3.2. 用户态线程

用户态线程也称作用户级线程（User Level Thread）。操作系统内核并不知道它的存在，它完全是在用户空间中创建。

用户级线程有很多优势，比如：

- 管理开销小：创建、销毁不需要系统调用。
- 切换成本低：用户空间程序可以自己维护，不需要走操作系统调度。

但是这种线程也有很多的缺点。

- 与内核协作成本高：比如这种线程完全是用户空间程序在管理，当它进行 I/O 的时候，无法利用到内核的优势，需要频繁进行用户态到内核态的切换。
- 线程间协作成本高：设想两个线程需要通信，通信需要 I/O，I/O 需要系统调用，因此用户态线程需要支付额外的系统调用成本。
- 无法利用多核优势：比如操作系统调度的仍然是这个线程所属的进程，所以无论每次一个进程有多少用户态的线程，都只能并发执行一个线程，因此一个进程的多个线程无法利用多核的优势。
- 操作系统无法针对线程调度进行优化：当一个进程的一个用户态线程阻塞（Block）了，操作系统无法及时发现和处理阻塞问题，它不会更换执行其他线程，从而造成资源浪费。

## 3.3. 内核态线程

内核态线程也称作内核级线程（Kernel Level Thread）。这种线程执行在内核态，可以通过系统调用创造一个内核级线程。

内核级线程有很多优势。

- 可以利用多核 CPU 优势：内核拥有较高权限，因此可以在多个 CPU 核心上执行内核线程。
- 操作系统级优化：内核中的线程操作 I/O 不需要进行系统调用；一个内核线程阻塞了，可以立即让另一个执行。

当然内核线程也有一些缺点。

- 创建成本高：创建的时候需要系统调用，也就是切换到内核态。
- 扩展性差：由一个内核程序管理，不可能数量太多。
- 切换成本较高：切换的时候，也同样存在需要内核操作，需要切换内核态

## 3.4. 用户态线程和内核态线程之间的映射关系

线程简单理解，就是要执行一段程序。程序不会自发的执行，需要操作系统进行调度。我们思考这样一个问题，如果有一个用户态的进程，它下面有多个线程。如果这个进程想要执行下面的某一个线程，应该如何做呢？

这时，比较常见的一种方式，就是将需要执行的程序，让一个内核线程去执行。毕竟，内核线程是真正的线程。因为它会分配到 CPU 的执行资源。

如果一个进程所有的线程都要自己调度，相当于在进程的主线程中实现分时算法调度每一个线程，也就是所有线程都用操作系统分配给主线程的时间片段执行。这种做法，相当于操作系统调度进程的主线程；进程的主线程进行二级调度，调度自己内部的线程。

这样操作劣势非常明显，比如无法利用多核优势，每个线程调度分配到的时间较少，而且这种线程在阻塞场景下会直接交出整个进程的执行权限。

由此可见，**用户态线程创建成本低，问题明显，不可以利用多核。内核态线程，创建成本高，可以利用多核，切换速度慢。因**此通常我们会在内核中预先创建一些线程，并反复利用这些线程。这样，用户态线程和内核态线程之间就构成了下面 4 种可能的关系：

- **多对一（Many to One）**

用户态进程中的多线程复用一个内核态线程。这样，极大地减少了创建内核态线程的成本，但是线程不可以并发。因此，这种模型现在基本上用的很少。

用户态线程怎么用内核态线程执行程序？程序是存储在内存中的指令，用户态线程是可以准备好程序让内核态线程执行的。后面的几种方式也是利用这样的方法。

![](/assets/images/posts/linux-user-kernel-space/linux-user-kernel-space-6.png)

- **一对一（One to One）**

该模型为每个用户态的线程分配一个单独的内核态线程，在这种情况下，每个用户态都需要通过系统调用创建一个绑定的内核线程，并附加在上面执行。 这种模型允许所有线程并发执行，能够充分利用多核优势，Windows NT 内核采取的就是这种模型。但是因为线程较多，对内核调度的压力会明显增加。

![](/assets/images/posts/linux-user-kernel-space/linux-user-kernel-space-7.png)

- **多对多（Many To Many）**

这种模式下会为 n 个用户态线程分配 m 个内核态线程。m 通常可以小于 n。一种可行的策略是将 m 设置为核数。这种多对多的关系，减少了内核线程，同时也保证了多核心并发。Linux 目前采用的就是该模型。

![](/assets/images/posts/linux-user-kernel-space/linux-user-kernel-space-8.png)

- **两层设计（Two Level）**

这种模型混合了多对多和一对一的特点。多数用户态线程和内核线程是 n 对 m 的关系，少量用户线程可以指定成 1 对 1 的关系。

![](/assets/images/posts/linux-user-kernel-space/linux-user-kernel-space-9.png)

# 4. JVM线程是用户态还是内核态

绿色线程：在计算机程序设计中，**绿色线程**是一种由运行环境或虚拟机(VM)调度，而不是由本地底层操作系统调度的线程。绿色线程并不依赖底层的系统功能，模拟实现了多线程的运行，这种线程的管理调配发生在用户空间而不是内核空间，所以它们可以在没有原生线程支持的环境中工作。

在Java 1.1中，绿色线程（至少在 Solaris 上）是JVM 中使用的唯一一种线程模型。 由于绿色线程和原生线程比起来在使用时有一些限制，操作系统只调度内核级线程，用户级线程相当于基于操作系统分配到进程主线程的时间片，再次拆分，因此无法利用多核特性。随后的 Java 版本中放弃了绿色线程，转而使用原生线程。在 Windows 上是 1 对 1 的模型，在 Linux 上是 n 对 m 的模型

https://docs.oracle.com/cd/E19620-01/805-4031/6j3qv1oej/index.html

> Green threads are threads implemented at the application level rather than in the OS. This is usually done when the OS does not provide a  thread API, or it doesn't work the way you need. 
>
> Thus, the advantage is that you get thread-like functionality at all. The disadvantage is that green threads can't actually use multiple  cores.
>
> There were a few early JVMs that used green threads (IIRC the  Blackdown JVM port to Linux did), but nowadays all mainstream JVMs use  real threads. There may be some embedded JVMs that still use green  threads.
>
> https://stackoverflow.com/questions/5713142/green-threads-vs-non-green-threads

> > Does every thread create their own instance of the JVM to handle their particular execution?
>
> No.  They execute in the same JVM so that (for example) they can share objects and class attributes. 
>
> ------
>
> > If not then does the JVM have to have some way to schedule which thread it will handle next
>
> There are two kinds of thread implementation in Java.  Native threads are mapped onto a thread abstraction which is implemented by the host  OS.  The OS takes care of native thread scheduling, and time slicing.  
>
> The second kind of thread is "green threads".  These are implemented  and managed by the JVM itself, with the JVM implementing thread  scheduling.  Java green thread implementations have not been supported  by Sun / Oracle JVMs since Java 1.2.
>
> https://stackoverflow.com/questions/2653458/understanding-javas-native-threads-and-the-jvm

# 5. 参考资料

http://www.ruanyifeng.com/blog/2016/12/user_space_vs_kernel_space.html

https://blog4jimmy.com/2018/01/348.html

https://cllc.fun/2019/03/02/linux-user-kernel-space/

https://mp.weixin.qq.com/s/mZujKx1bKl1T6gEI1s400Q

《重学操作系统 》