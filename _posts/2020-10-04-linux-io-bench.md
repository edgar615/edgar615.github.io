---
layout: post
title: 性能排查 - IO基准测试
date: 2020-10-04
categories:
    - linux
comments: true
permalink: linux-io-bench.html
---



```
# 安装
apt-get install -y fio

# 随机读
fio -name=randread -direct=1 -iodepth=64 -rw=randread -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb

# 随机写
fio -name=randwrite -direct=1 -iodepth=64 -rw=randwrite -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb

# 顺序读
fio -name=read -direct=1 -iodepth=64 -rw=read -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb

# 顺序写
fio -name=write -direct=1 -iodepth=64 -rw=write -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb 
```

- direct，表示是否跳过系统缓存。1 表示跳过系统缓存。
- iodepth，表示使用异步 I/O（asynchronous I/O，简称 AIO）时，同时发出的 I/O 请求上限。
- rw，表示 I/O 模式。read/write 分别表示顺序读 / 写，而 randread/randwrite 则分别表示随机读 / 写。
- ioengine，表示 I/O 引擎，它支持同步（sync）、异步（libaio）、内存映射（mmap）、网络（net）等各种 I/O 引擎。
- bs，表示 I/O 的大小。
- filename，表示文件路径，当然，它可以是磁盘路径（测试磁盘性能），也可以是文件路径（测试文件系统性能）。

输出结果

```
randread: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=64
fio-3.16
Starting 1 process
randread: Laying out IO file (1 file / 1024MiB)
Jobs: 1 (f=1): [r(1)][100.0%][r=80.4MiB/s][r=20.6k IOPS][eta 00m:00s]
randread: (groupid=0, jobs=1): err= 0: pid=3240: Fri Jan 22 06:06:06 2021
  read: IOPS=21.4k, BW=83.5MiB/s (87.5MB/s)(1024MiB/12265msec)
    slat (nsec): min=1703, max=19738k, avg=29778.53, stdev=82206.59
    clat (usec): min=35, max=38136, avg=2962.91, stdev=1552.59
     lat (usec): min=102, max=38139, avg=2992.94, stdev=1545.97
    clat percentiles (usec):
     |  1.00th=[  594],  5.00th=[ 1401], 10.00th=[ 1893], 20.00th=[ 2089],
     | 30.00th=[ 2212], 40.00th=[ 2409], 50.00th=[ 2638], 60.00th=[ 2966],
     | 70.00th=[ 3294], 80.00th=[ 3687], 90.00th=[ 4424], 95.00th=[ 5211],
     | 99.00th=[ 6915], 99.50th=[ 8586], 99.90th=[22676], 99.95th=[26608],
     | 99.99th=[34866]
   bw (  KiB/s): min=66704, max=110642, per=99.74%, avg=85273.04, stdev=13110.59, samples=24
   iops        : min=16676, max=27660, avg=21318.04, stdev=3277.65, samples=24
  lat (usec)   : 50=0.01%, 100=0.01%, 250=0.23%, 500=0.54%, 750=0.66%
  lat (usec)   : 1000=0.95%
  lat (msec)   : 2=10.72%, 4=72.19%, 10=14.30%, 20=0.28%, 50=0.13%
  cpu          : usr=0.98%, sys=70.87%, ctx=12725, majf=0, minf=75
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.1%, >=64=0.0%
     issued rwts: total=262144,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=64

Run status group 0 (all jobs):
   READ: bw=83.5MiB/s (87.5MB/s), 83.5MiB/s-83.5MiB/s (87.5MB/s-87.5MB/s), io=1024MiB (1074MB), run=12265-12265msec

Disk stats (read/write):
    dm-0: ios=261444/11, merge=0/0, ticks=483212/20, in_queue=483232, util=99.38%, aggrios=261469/4, aggrmerge=675/7, aggrticks=479358/4, aggrin_queue=101096, aggrutil=99.00%
  sda: ios=261469/4, merge=675/7, ticks=479358/4, in_queue=101096, util=99.00%

```

- slat ，是指从 I/O 提交到实际执行 I/O 的时长（Submission latency）；
- clat ，是指从 I/O 提交到 I/O 完成的时长（Completion latency）；
- lat ，指的是从 fio 创建 I/O 到 I/O 完成的总时长
- bw ，指的是吞吐量
- iops ，指的是每秒 I/O 的次数

对同步 I/O 来说，由于 I/O 提交和 I/O 完成是一个动作，所以 slat 实际上就是 I/O 完成的时间，而 clat 是 0。使用异步 I/O（libaio）时，lat 近似等于 slat + clat 之和。

应用程序的 I/O 都是读写并行的，而且每次的 I/O 大小也不一定相同。所以，刚刚说的这几种场景，并不能精确模拟应用程序的 I/O 模式

fio 支持 I/O 的重放。借助 blktrace，再配合上 fio，就可以实现对应用程序 I/O 模式的基准测试。你需要先用 blktrace ，记录磁盘设备的 I/O 访问情况；然后使用 fio ，重放 blktrace 的记录。

```

# 使用blktrace跟踪磁盘I/O，注意指定应用程序正在操作的磁盘
$ blktrace /dev/sdb

# 查看blktrace记录的结果
# ls
sdb.blktrace.0  sdb.blktrace.1

# 将结果转化为二进制文件
$ blkparse sdb -d sdb.bin

# 使用fio重放日志
$ fio --name=replay --filename=/dev/sdb --direct=1 --read_iolog=sdb.bin 
```

