---
layout: post
title: Linux监控工具-vmstat
date: 2019-10-30
categories:
    - linux
comments: true
permalink: linux-monitor-vmstat.html
---

vmstat 是一个常用的系统性能分析工具，主要用来分析系统的内存使用情况，也常用来分析 CPU 上下文切换和中断的次数。

下面就是一个 vmstat 的使用示例：

```
# vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 2618268  71056 616608    0    0    82     9   58  100  0  0 100  0  0
```

`procs`：procs 中有 `r` 和 `b` 列，它报告进程统计信息。在上面的输出中，在运行队列（`r`）中有1个进程在等待 CPU 并有零个休眠进程（`b`）。通常，它不应该超过处理器（或核心）的数量，如果你发现异常，最好使用 top 命令进一步地排除故障。

- `r`：等待运行的进程数。
- `b`：休眠状态下的进程数。

`memory`： memory 下有报告内存统计的 `swpd`、`free`、`buff` 和 `cache` 列。你可以用 `free -m` 命令看到同样的信息。在上面的内存统计中，统计数据以千字节表示，这有点难以理解，最好添加 `M` 参数来看到以兆字节为单位的统计数据。

- `swpd`：使用的虚拟内存量。
- `free`：空闲内存量。
- `buff`：用作缓冲区的内存量。
- `cache`：用作高速缓存的内存量。

`swap`：swap 有 `si` 和 `so` 列，用于报告交换内存统计信息。你可以用 `free -m` 命令看到相同的信息。

- `si`：从磁盘交换的内存量（换入，从 swap 移到实际内存的内存）。
- `so`：交换到磁盘的内存量（换出，从实际内存移动到 swap 的内存）。

`I/O`：I/O 有 `bi` 和 `bo` 列，它以“块读取”和“块写入”的单位来报告每秒磁盘读取和写入的块的统计信息。如果你发现有巨大的 I/O 读写，最好使用 iotop 和 iostat 命令来查看。

- `bi`：从块设备接收的块数。
- `bo`：发送到块设备的块数。

`system`：system 有 `in` 和 `cs` 列，它报告每秒的系统操作。

- `in`：每秒的系统中断数，包括时钟中断。
- `cs`：每秒上下文切换的次数。

`CPU`：CPU 有 `us`、`sy`、`id` 和 `wa` 列，报告（所用的） CPU 资源占总 CPU 时间的百分比。如果你发现异常，最好使用 `top` 和 `free` 命令。

- `us`：处理器在非内核程序消耗的时间。
- `sy`：处理器在内核相关任务上消耗的时间。
- `id`：处理器的空闲时间。
- `wa`：处理器在等待IO操作完成以继续处理任务上的时间。

**以 MB 方式输出**

```
# vmstat -S M
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0   2556     69    602    0    0    73     8   58   98  0  0 100  0  0
```

**将每 2 秒运行一次**

```
# vmstat 2
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 2617372  71760 616824    0    0    65     8   57   97  0  0 100  0  0
 0  0      0 2617164  71760 616824    0    0     0     0  195  321  0  0 100  0  0
 0  0      0 2617416  71768 616816    0    0     0    10  208  350  0  0 100  0  0
 0  0      0 2617416  71768 616824    0    0     0     0  198  334  0  0 100  0  0
```

**每 2 秒运行一次，5次后自动退出**

```
# vmstat 2 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 2617424  71824 616832    0    0    64     8   57   97  0  0 100  0  0
 0  0      0 2617384  71832 616824    0    0     0    16  213  347  0  0 100  0  0
 0  0      0 2617384  71832 616832    0    0     0     0  199  329  0  0 100  0  0
 0  0      0 2617384  71832 616832    0    0     0     0  188  325  0  0 100  0  0
 0  0      0 2617384  71840 616832    0    0     0    10  209  347  0  0 100  0  0
```

**显示活动和非活动内存**

```
# vmstat -a
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free  inact active   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 2617140 406436 668720    0    0    63     8   57   96  0  0 100  0  0
```

**打印磁盘统计**

```
# vmstat -d
disk- ------------reads------------ ------------writes----------- -----IO------
       total merged sectors      ms  total merged sectors      ms    cur    sec
loop0     41      0     674       9      0      0       0       0      0      0
loop1   1442      0    3476     801      0      0       0       0      0      0
loop2     68      0    2098      26      0      0       0       0      0      0
loop3     43      0     672       9      0      0       0       0      0      0
loop4     56      0    2108      20      0      0       0       0      0      0
loop5  13118      0   26840    5870      0      0       0       0      0      1
loop6      4      0       8       0      0      0       0       0      0      0
loop7      0      0       0       0      0      0       0       0      0      0
fd0       23      0     184     798      0      0       0       0      0      0
sr0       81      0    6288      16      0      0       0       0      0      0
sr1        0      0       0       0      0      0       0       0      0      0
sda    14846   5637 1046572    5055   4955   4590  132304    1368      0      8
dm-0   20191      0 1029410    8264   9491      0  131976    1652      0      8
```

**总结磁盘统计**

```
# vmstat -D
           13 disks
            3 partitions
        49913 total reads
         5637 merged reads
      2118330 read sectors
        20868 milli reading
        14580 writes
         4618 merged writes
       265656 written sectors
         3032 milli writing
            0 inprogress IO
           17 milli spent IO
```

**打印更多统计**

```
# vmstat -D
           13 disks
            3 partitions
        49913 total reads
         5637 merged reads
      2118330 read sectors
        20868 milli reading
        14580 writes
         4618 merged writes
       265656 written sectors
         3032 milli writing
            0 inprogress IO
           17 milli spent IO
root@server-1:~# vmstat -s
      4001736 K total memory
       695760 K used memory
       668980 K active memory
       406576 K inactive memory
      2616880 K free memory
        72120 K buffer memory
       616976 K swap cache
      4001788 K total swap
            0 K used swap
      4001788 K free swap
          435 non-nice user cpu ticks
            8 nice user cpu ticks
         1764 system cpu ticks
       919958 idle cpu ticks
          128 IO-wait cpu ticks
            0 IRQ cpu ticks
           79 softirq cpu ticks
            0 stolen cpu ticks
       544460 pages paged in
        67108 pages paged out
            0 pages swapped in
            0 pages swapped out
       517848 interrupts
       882438 CPU context switches
   1611119425 boot time
         2735 forks
```



