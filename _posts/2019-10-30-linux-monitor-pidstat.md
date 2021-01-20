---
layout: post
title: Linux监控工具-pidstat
date: 2019-10-30
categories:
    - linux
comments: true
permalink: linux-monitor-pidstat.html
---

pidstat是sysstat工具的一个命令，用于监控全部或指定进程的cpu、内存、线程、设备IO等系统资源的占用情况。pidstat首次运行时显示自系统启动开始的各项统计信息，之后运行pidstat将显示自上次运行该命令以后的统计信息。用户可以通过指定统计的次数和时间来获得所需的统计信息。

**安装**

```
apt-get install sysstat
```

**用法**

```
pidstat [ 选项 ] [ <时间间隔> ] [ <次数> ]
```

常用的参数：

- -u：默认的参数，显示各个进程的cpu使用统计
- -r：显示各个进程的内存使用统计
- -d：显示各个进程的IO使用情况
- -p：指定进程号
- -w：显示每个进程的上下文切换情况
- -t：显示选择任务的线程的统计信息外的额外信息
- -T { TASK | CHILD | ALL } 这个选项指定了pidstat监控的。TASK表示报告独立的task，CHILD关键字表示报告进程下所有线程统计信息。ALL表示报告独立的task和task下面的所有线程。
   注意：task和子线程的全局的统计信息和pidstat选项无关。这些统计信息不会对应到当前的统计间隔，这些统计信息只有在子线程kill或者完成的时候才会被收集。
- -V：版本号
- -h：在一行上显示了所有活动，这样其他程序可以容易解析。
- -I：在SMP环境，表示任务的CPU使用率/内核数量
- -l：显示命令名和所有参数

**查看所有进程的  CPU 使用情况**

```
# pidstat
Linux 5.4.0-62-generic (server-1)       01/20/2021      _x86_64_        (4 CPU)

07:57:20 AM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
07:57:20 AM     0         1    0.00    0.02    0.00    0.00    0.03     1  systemd
07:57:20 AM     0         2    0.00    0.00    0.00    0.00    0.00     1  kthreadd
07:57:20 AM     0        10    0.00    0.00    0.00    0.00    0.00     0  ksoftirqd/0
```

- PID：进程ID
- %usr：进程在用户空间占用cpu的百分比
- %system：进程在内核空间占用cpu的百分比
- %guest：进程在虚拟机占用cpu的百分比
- %CPU：进程占用cpu的百分比
- CPU：处理进程的cpu编号
- Command：当前进程对应的命令

**cpu使用情况统计**

```
# pidstat -u
Linux 5.4.0-62-generic (server-1)       01/20/2021      _x86_64_        (4 CPU)

07:59:02 AM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
07:59:02 AM     0         1    0.00    0.02    0.00    0.00    0.03     1  systemd
07:59:02 AM     0         2    0.00    0.00    0.00    0.00    0.00     3  kthreadd
07:59:02 AM     0        10    0.00    0.00    0.00    0.00    0.00     0  ksoftirqd/0
07:59:02 AM     0        11    0.00    0.08    0.00    0.02    0.08     2  rcu_sched

```

**内存使用情况统计**

```
# pidstat -r
Linux 5.4.0-62-generic (server-1)       01/20/2021      _x86_64_        (4 CPU)

07:59:37 AM   UID       PID  minflt/s  majflt/s     VSZ     RSS   %MEM  Command
07:59:37 AM     0         1      1.66      0.01  168824   12744   0.32  systemd
07:59:37 AM     0       500      1.92      0.00   43444   20980   0.52  systemd-journal
07:59:37 AM     0       528      1.08      0.00   21360    5536   0.14  systemd-udevd
07:59:37 AM     0       713      0.30      0.00  345796   18196   0.45  multipathd

```

- PID：进程标识符
- Minflt/s:任务每秒发生的次要错误，不需要从磁盘中加载页
- Majflt/s:任务每秒发生的主要错误，需要从磁盘中加载页
- VSZ：虚拟地址大小，虚拟内存的使用KB
- RSS：常驻集合大小，非交换区五里内存使用KB
- Command：task命令名

**显示各个进程的IO使用情况**

```
# pidstat -d
Linux 5.4.0-62-generic (server-1)       01/20/2021      _x86_64_        (4 CPU)

08:01:00 AM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
08:01:00 AM     0         1     33.67     18.46     13.46      12  systemd
08:01:00 AM     0       430      0.00      5.59      0.00      37  jbd2/dm-0-8
08:01:00 AM     0       500      0.14      6.73      0.00       2  systemd-journal
08:01:00 AM     0       528      3.45      0.00      0.00       0  systemd-udevd

```

- PID：进程id
- kB_rd/s：每秒从磁盘读取的KB
- kB_wr/s：每秒写入磁盘KB
- kB_ccwr/s：任务取消的写入磁盘的KB。当任务截断脏的pagecache的时候会发生。
- COMMAND:task的命令名

**显示各个进程的上下文切换情况**

```
# pidstat -w
Linux 5.4.0-62-generic (server-1)       01/20/2021      _x86_64_        (4 CPU)

08:01:58 AM   UID       PID   cswch/s nvcswch/s  Command
08:01:58 AM     0         1      0.19      0.26  systemd
08:01:58 AM     0         2      0.03      0.00  kthreadd
08:01:58 AM     0         3      0.00      0.00  rcu_gp

```

- PID:进程id
- Cswch/s:每秒主动任务上下文切换数量
- Nvcswch/s:每秒被动任务上下文切换数量
- Command:命令名

**显示选择任务的线程的统计信息外的额外信息**

```
# pidstat -t
Linux 5.4.0-62-generic (server-1)       01/20/2021      _x86_64_        (4 CPU)

08:03:00 AM   UID      TGID       TID    %usr %system  %guest   %wait    %CPU   CPU  Command
08:03:00 AM     0         1         -    0.00    0.02    0.00    0.00    0.03     1  systemd
08:03:00 AM     0         -         1    0.00    0.02    0.00    0.00    0.03     1  |__systemd
08:03:00 AM     0         2         -    0.00    0.00    0.00    0.00    0.00     3  kthreadd
08:03:00 AM     0         -         2    0.00    0.00    0.00    0.00    0.00     3  |__kthreadd
08:03:00 AM     0        10         -    0.00    0.00    0.00    0.00    0.00     0  ksoftirqd/0
08:03:00 AM     0         -        10    0.00    0.00    0.00    0.00    0.00     0  |__ksoftirqd/0
```

- TGID:主线程的表示
- TID:线程id
- %usr：进程在用户空间占用cpu的百分比
- %system：进程在内核空间占用cpu的百分比
- %guest：进程在虚拟机占用cpu的百分比
- %CPU：进程占用cpu的百分比
- CPU：处理进程的cpu编号
- Command：当前进程对应的命令

