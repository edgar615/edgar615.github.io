---
layout: post
title: 性能排查 - IO
date: 2020-10-03
categories:
    - linux
comments: true
permalink: linux-trouble-io.html
---

# 1. 指标

存储空间的使用情况，包括容量、使用量以及剩余空间

使用率，是指磁盘处理 I/O 的时间百分比。过高的使用率（比如超过 80%），通常意味着磁盘 I/O 存在性能瓶颈。使用率只考虑有没有 I/O，而不考虑 I/O 的大小。换句话说，当使用率是 100% 的时候，磁盘依然有可能接受新的 I/O 请求。

饱和度，是指磁盘处理 I/O 的繁忙程度。过高的饱和度，意味着磁盘存在严重的性能瓶颈。当饱和度为 100% 时，磁盘无法接受新的 I/O 请求。

IOPS（Input/Output Per Second），是指每秒的 I/O 请求数。

吞吐量，是指每秒的 I/O 请求大小。

响应时间，是指 I/O 请求从发出到收到响应的间隔时间。

# 2. **排查工具**

![](/assets/images/posts/linux-trouble-io/linux-trouble-io-1.png)

![](/assets/images/posts/linux-trouble-io/linux-trouble-io-2.jpg)

# 3. 排查过程

- 先用 iostat 发现磁盘 I/O 性能瓶颈；
- 再借助 pidstat ，定位出导致瓶颈的进程；
- 随后分析进程的 I/O 行为；
- 最后，结合应用程序的原理，分析这些 I/O 的来源。

![](/assets/images/posts/linux-trouble-io/linux-trouble-io-3.png)

# 4. 日志过多

通过top发现cpu的iowait略高，而buff/cache占用了大部分内存

```
top - 05:19:51 up  2:04,  2 users,  load average: 0.88, 0.29, 0.10
Tasks: 234 total,   2 running, 232 sleeping,   0 stopped,   0 zombie
%Cpu0  :  0.4 us,  5.7 sy,  0.0 ni, 84.0 id,  6.8 wa,  0.0 hi,  3.2 si,  0.0 st
%Cpu1  :  0.3 us,  8.6 sy,  0.0 ni, 89.7 id,  1.4 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu2  : 15.9 us, 69.6 sy,  0.0 ni, 10.1 id,  4.4 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu3  :  0.7 us, 10.8 sy,  0.0 ni, 86.5 id,  1.7 wa,  0.0 hi,  0.3 si,  0.0 st
MiB Mem :   3907.9 total,    830.7 free,   1149.8 used,   1927.4 buff/cache
MiB Swap:   3908.0 total,   3902.7 free,      5.3 used.   2491.2 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   6137 root      20   0  963344 934344   5352 R  89.7  23.3   3:04.23 python
```

通过pidstat分析进程IO，可以发现Python进程美妙写入300M的数据，而且也存在延迟

```
# pidstat -d 1
Linux 5.4.0-62-generic (server-1)       01/22/2021      _x86_64_        (4 CPU)

05:26:09 AM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
05:26:10 AM     0      6137      0.00 304162.38      0.00       0  python

05:26:10 AM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
05:26:11 AM     0       431      0.00    172.00      0.00       0  jbd2/dm-0-8
05:26:11 AM     0       500      0.00    152.00      0.00       0  systemd-journal
05:26:11 AM   104       872      0.00      4.00      0.00       0  rsyslogd
05:26:11 AM     0      4300      4.00      0.00      0.00      57  kworker/u256:1-events_power_efficient
05:26:11 AM     0      6137      0.00 307200.00      0.00      23  python

05:26:11 AM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
05:26:12 AM     0      6137      0.00 307200.00      0.00       0  python
05:26:12 AM     0      6351      0.00      0.00      0.00      20  kworker/u256:2-events_power_efficient

05:26:12 AM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
05:26:13 AM     0      6137      0.00 307204.00      0.00       0  python

```

> kworker 是一个内核线程，而 jbd2 是 ext4 文件系统中，用来保证数据完整性的内核线程。他们都是保证文件系统基本功能的内核线程，它们延迟的根源也是大量 I/O。

strace 命令，分析python的系统调用

```
# strace -p 6137
strace: Process 6137 attached
mremap(0x7f2839af5000, 393220096, 314576896, MREMAP_MAYMOVE) = 0x7f2839af5000
munmap(0x7f28511f6000, 314576896)       = 0
lseek(3, 0, SEEK_END)                   = 629145690
lseek(3, 0, SEEK_CUR)                   = 629145690
munmap(0x7f2839af5000, 314576896)       = 0
mmap(NULL, 314576896, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f28511f6000
mmap(NULL, 314576896, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f283e5f5000
write(3, "2021-01-22 05:30:36,380 - __main"..., 314572844) = 314572844
munmap(0x7f283e5f5000, 314576896)       = 0
write(3, "\n", 1)                       = 1
munmap(0x7f28511f6000, 314576896)       = 0
select(0, NULL, NULL, NULL, {tv_sec=0, tv_usec=100000}) = 0 (Timeout)
getpid()                                = 1
mmap(NULL, 314576896, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f28511f6000
mmap(NULL, 393220096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f2839af5000
mremap(0x7f2839af5000, 393220096, 314576896, MREMAP_MAYMOVE) = 0x7f2839af5000
munmap(0x7f28511f6000, 314576896)       = 0
lseek(3, 0, SEEK_END)                   = 943718535
lseek(3, 0, SEEK_CUR)                   = 943718535
munmap(0x7f2839af5000, 314576896)       = 0
close(3)                                = 0
stat("/tmp/logtest.txt.1", {st_mode=S_IFREG|0644, st_size=943718535, ...}) = 0
```

在write调用上可以看到，进程向文件描述符编号为 3 的文件中，写入了 600MB 的数据。

再观察后面的 stat() 调用，你可以看到，它正在获取 /tmp/logtest.txt.1 的状态

使用 lsof 命令查看Python进程打开了哪些文件

```
# lsof -p 6137
COMMAND  PID USER   FD   TYPE DEVICE  SIZE/OFF    NODE NAME
python  6137 root  cwd    DIR   0,53      4096  540700 /
python  6137 root  rtd    DIR   0,53      4096  540700 /
python  6137 root  txt    REG   0,53     28016 2376697 /usr/local/bin/python3.7
...
python  6137 root    0u   CHR  136,0       0t0       3 /dev/pts/0
python  6137 root    1u   CHR  136,0       0t0       3 /dev/pts/0
python  6137 root    2u   CHR  136,0       0t0       3 /dev/pts/0
python  6137 root    3w   REG  253,0 629145690 1572873 /tmp/logtest.txt
```

可以看到/tmp/logtest.txt，它的文件描述符是 3 号，而 3 后面的 w ，表示以写的方式打开

下面就需要去检查我们的程序为什么会狂写/tmp/logtest.txt了

# 5.  MySQL慢查询

通过top、iostat、pidstat找到大量IO的进程，通过lsof定位到文件，通过show full processlist;定位到sql



# 6. 参考资料

《Linux性能优化实战》