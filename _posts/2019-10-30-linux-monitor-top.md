---
layout: post
title: Linux监控工具-top
date: 2019-10-30
categories:
    - linux
comments: true
permalink: linux-monitor-top.html
---

top命令是Linux下常用的性能分析工具，能够实时显示系统中各个进程的资源占用状况，类似于Windows的任务管理器。top是一个动态显示过程,即可以通过用户按键来不断刷新当前状态.如果在前台执行该命令,它将独占前台,直到用户终止该程序为止.比较准确的说,top命令提供了实时的对系统处理器的状态监视.它将显示系统中CPU最“敏感”的任务列表.该命令可以按CPU使用.内存使用和执行时间对任务进行排序；而且该命令的很多特性都可以通过交互式命令或者在个人定制文件中进行设定。


语法
```
[root@ihorn-dev ~]# top -h
  procps-ng version 3.3.9
Usage:
  top -hv | -bcHiOSs -d secs -n max -u|U user -p pid(s) -o field -w [cols]
```
参数
```
-b：以批处理模式操作； 
-c：显示完整的治命令； 
-d：屏幕刷新间隔时间； 
-I：忽略失效过程； 
-s：保密模式； 
-S：累积模式； 
-i<时间>：设置间隔时间； 
-u<用户名>：指定用户名； 
-p<进程号>：指定进程； 
-n<次数>：循环显示的次数。
```
交互命令
```
h：显示帮助画面，给出一些简短的命令总结说明；
k：终止一个进程； 
i：忽略闲置和僵死进程，这是一个开关式命令； 
q：退出程序； 
r：重新安排一个进程的优先级别； 
S：切换到累计模式； 
s：改变两次刷新之间的延迟时间（单位为s），如果有小数，就换算成ms。输入0值则系统将不断刷新，默认值是5s； 
f或者F：从当前显示中添加或者删除项目； 
o或者O：改变显示项目的顺序； 
l：切换显示平均负载和启动时间信息；
m：切换显示内存信息； 
t：切换显示进程和CPU状态信息； 
c：切换显示命令名称和完整命令行； 
M：根据驻留内存大小进行排序； 
P：根据CPU使用百分比大小进行排序； 
T：根据时间/累计时间进行排序； 
w：将当前设置写入~/.toprc文件中。
```

示例：
```
[root@ihorn-dev]# top
top - 14:58:36 up 12 days,  3:30,  5 users,  load average: 0.09, 0.11, 0.19
Tasks: 194 total,   1 running, 193 sleeping,   0 stopped,   0 zombie
%Cpu(s):  1.8 us,  0.8 sy,  0.0 ni, 96.9 id,  0.4 wa,  0.0 hi,  0.1 si,  0.0 st
KiB Mem:  16269468 total, 10890328 used,  5379140 free,   366196 buffers
KiB Swap:        0 total,        0 used,        0 free.  3264548 cached Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                                              
11718 root      20   0 1215464  72080   9988 S   5.0  0.4 662:01.56 node /alidata/s                                                                                                                      
 2636 root      20   0 5508804 1.155g  11220 S   2.0  7.4   2538:52 java                                                                                                                                 
 3483 root      20   0 4076364 615016  15572 S   1.7  3.8  10:51.64 java                                                                                                                                 
 1410 root      20   0    1544    576    500 S   0.7  0.0   8:46.74 aliyun-service                                                                                                                       
 3577 root      20   0 3988360 598268  15708 S   0.7  3.7   9:39.25 java                                                                                                                                 
 3783 root      20   0 3925748 360156  15724 S   0.7  2.2  51:17.43 java                                                                                                                                 
 3924 root      20   0 3918884 336540  15252 S   0.7  2.1   1:11.29 java                                                                                                                                 
15040 root      20   0 3934708 372284  11760 S   0.7  2.3 366:27.80 java
```
第一行(top...)

```
top - 14:58:36 up 12 days,  3:30,  5 users,  load average: 0.09, 0.11, 0.19
```

- 14:58:36：系统当前时间
- up 12 days,  3:30： 系统开机到现在经过了多少时间
- 5 users ：当前5用户在线
- load average: 0.09, 0.11, 0.19：系统1分钟、5分钟、15分钟的CPU负载信息。越小代表系统余额空闲

第二行(Tasks...)：

```
Tasks: 194 total,   1 running, 193 sleeping,   0 stopped,   0 zombie
```

显示的是目前程序的总量，以及每个状态的程序总量(running, sleeping, stopped, zombie)。 

- 194 total：就是当前有194个任务，也就是194个进程
- 1 running：1个进程正在运行
- 193 sleeping：193个进程睡眠
- 0 stopped：停止的进程数
- 0 zombie：僵死的进程数，如果这个数组不是0，需要检查一下为什么进程会变成僵尸进程

第三行(Cpus...)：

```
%Cpu(s):  1.8 us,  0.8 sy,  0.0 ni, 96.9 id,  0.4 wa,  0.0 hi,  0.1 si,  0.0 st
```

显示的是 CPU 的整体负载

- 1.8 us, user： 用户态进程(未调整优先级的) 占用CPU时间百分比

- 0.7 sy，system: 内核占用CPU时间百分比
- 0.0 ni，niced：改变过优先级的用户态进程占用CPU的百分比
- 99.3 id：空闲CPU时间百分比
- 0.4 wa，IO wait：等待I/O的CPU时间百分比，通常系统会变慢都是I/0产生的原因比较大
- 0.0 hi：处理硬件中断的CPU时间
- 0.1 si: 处理软件中断的CPU时间
- 0.0 st：CPU软中断时间百分比

这里显示数据是所有cpu的平均值，如果想看每一个cpu的处理情况，按1即可；折叠，再次按1；


```
top - 15:04:10 up 12 days,  3:36,  5 users,  load average: 0.07, 0.13, 0.18
Tasks: 195 total,   1 running, 194 sleeping,   0 stopped,   0 zombie
%Cpu0  :  2.7 us,  0.7 sy,  0.0 ni, 96.3 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
%Cpu1  :  3.0 us,  0.7 sy,  0.0 ni, 96.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu2  :  2.4 us,  0.3 sy,  0.0 ni, 97.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu3  :  4.4 us,  0.3 sy,  0.0 ni, 95.0 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
KiB Mem:  16269468 total, 10941652 used,  5327816 free,   366196 buffers
KiB Swap:        0 total,        0 used,        0 free.  3269396 cached Mem
```

第四行

```
KiB Mem:  16269468 total, 10890328 used,  5379140 free,   366196 buffers
```

显示物理内存的使用情况

- 16269468 total：物理内存总量
- 10890328 used：使用的物理内存量
- 5379140 free：空闲的物理内存量
- 366196 buffers：用作内核缓存的物理内存量

第五行

```
KiB Swap:        0 total,        0 used,        0 free.  3264548 cached Mem
```

显示虚拟内存（交换空间swap）的使用情况

- 0 total：交换区总量
- 0 used：使用的交换区量
- 0 free：空闲的交换区量
- 3264548cached：缓冲交换区总量

swap 的使用量要尽量的少！如果 swap 被用的很大量，表示系统的实体内存实在不足。

根据进程ID检查swap信息
```
    [root@ihorn-dev redis-3.2.0]# cat /proc/2721/smaps | grep Swap
    Swap:                  0 kB
    Swap:                  0 kB
    Swap:                  0 kB
    Swap:                  0 kB
```

**注意:**

- used这个统计量，并不是用户线程占用内存之和，而是Linux系统总共消耗的内存，用户线程是“寄生”于Linux OS之上的；
- 对于free统计量，是“干净的”可以直接分配的内存，但是cached和buffered这两种状态的内存，主要是映射到磁盘的缓冲，也能够被继续分配
- 真正被使用的内存real free memory ＝ used - buffers - cached；真正可用的内存real free memory = total - real free memory

使用free命令一样
```
[root@ihorn-product ~]# free -h
			 total       used       free     shared    buffers     cached
Mem:           15G        15G       189M       328M       274M       8.4G
-/+ buffers/cache:       6.7G       8.8G
Swap:           0B         0B         0B
```
第六行：这个是当在 top 程序当中输入命令时，显示状态的地方。

 top的下半部分显示了各个进程的详细信息。

```
PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                                              
11718 root      20   0 1215464  72080   9988 S   5.0  0.4 662:01.56 node /alidata/s                                                                                                                      
2636 root      20   0 5508804 1.155g  11220 S   2.0  7.4   2538:52 java                                                                                                                                 
3483 root      20   0 4076364 615016  15572 S   1.7  3.8  10:51.64 java                                                                                                                                 
1410 root      20   0    1544    576    500 S   0.7  0.0   8:46.74 aliyun-service   
```



- PID：进程的ID
- USER：进程所有者
- PR：进程的优先级别，越小越优先被执行
- NI：nice值。负值表示高优先级，正值表示低优先级，与 Priority 有关，也是越小越早被运行；
- VIRT：进程占用的虚拟内存
- RES：进程占用的物理内存
- SHR：进程使用的共享内存
- S：进程的状态。S表示休眠，R表示正在运行，Z表示僵死状态，N表示该进程优先值为负数
- %CPU：进程占用CPU的使用率
- %MEM：进程使用的物理内存和总内存的百分比
- TIME+：该进程启动后占用的总的CPU时间，即占用CPU使用时间的累加值。
- COMMAND：进程启动命令名称

### 更改显示内容

通过 f 键可以选择显示的内容。按 f 键之后会显示列的列表，按 a-z 即可显示或隐藏对应的列，最后按回车键确定。

按 o 键可以改变列的显示顺序。按小写的 a-z 可以将相应的列向右移动，而大写的 A-Z 可以将相应的列向左移动。最后按回车键确定。

按大写的 F 或 O 键，然后按 a-z 可以将进程按照相应的列进行排序。而大写的 R 键可以将当前的排序倒转

- a 	PID 	进程id
- b 	PPID 	父进程id
- c 	RUSER 	Real user name
- d 	UID 	进程所有者的用户id
- e 	USER 	进程所有者的用户名
- f 	GROUP 	进程所有者的组名
- g 	TTY 	启动进程的终端名。不是从终端启动的进程则显示为 ?
- h 	PR 	进程的调度优先级。这个字段的一些值是'rt'。这意味这这些进程运行在实时态。
- i 	NI 	进程的nice值（优先级）。越小的值意味着越高的优先级。
- j 	P 	最后使用的CPU，仅在多CPU环境下有意义
- k 	%CPU 	上次更新到现在的CPU时间占用百分比
- l 	TIME 	进程使用的CPU时间总计，单位秒
- m 	TIME+ 	任务启动后到现在所使用的全部CPU时间，精确到百分之一秒
- n 	%MEM 	进程使用的物理内存百分比
- o 	VIRT 	进程使用的虚拟内存总量，单位kb。VIRT=SWAP+RES
- p 	SWAP 	进程使用的虚拟内存中，被换出的大小，单位kb。
- q 	RES 	驻留内存大小。驻留内存是任务使用的非交换物理内存大小。单位kb。RES=CODE+DATA
- r 	CODE 	可执行代码占用的物理内存大小，单位kb
- s 	DATA 	可执行代码以外的部分(数据段+栈)占用的物理内存大小，单位kb
- t 	SHR 	进程使用的共享内存，单位kb
- u 	nFLT 	页面错误次数
- v 	nDRT 	最后一次写入到现在，被修改过的页面数。
- w 	S 	进程状态。	D=不可中断的睡眠状态	R=运行	S=睡眠	T=跟踪/停止	Z=僵尸进程
- x 	COMMAND 	运行进程所使用的命令
- y 	WCHAN 	若该进程在睡眠，则显示睡眠中的系统函数名
- z 	Flags 	任务标志，参考 sched.h



# 常用交互指令

- q：退出top命令
- <Space>：立即刷新
- s：设置刷新时间间隔
- c：显示命令完全模式
- t:：显示或隐藏进程和CPU状态信息
- m：显示或隐藏内存状态信息
- l：显示或隐藏uptime信息
- f：增加或减少进程显示标志
- S：累计模式，会把已完成或退出的子进程占用的CPU时间累计到父进程的MITE+
- P：按%CPU使用率排行
- T：按MITE+排行
- M：按%MEM排行
- u：指定显示用户进程
- r：修改进程renice值
- kkill：进程
- i：只显示正在运行的进程
- W：保存对top的设置到文件^/.toprc，下次启动将自动调用toprc文件的设置。
- h：帮助命令。
- q：退出

top命令默认在一个特定间隔(3秒)后刷新显示。要手动刷新，用户可以输入回车或者空格。

# 小技巧
## 高亮显示当前运行进程
按下键盘“b”（打开/关闭加亮效果），当按下后，当前正在运行的进程就会被高亮显示

示例：将 top 的资讯进行 2 次，然后将结果输出到 /tmp/top.txt
```
[root@www ~]# top -b -n 2 > /tmp/top.txt
```

## 进程字段排序
默认进入top时，各进程是按照CPU的占用量来排序的。但是，我们可以改变这种排序：

-  M：根据驻留内存大小进行排序
- P：根据CPU使用百分比大小进行排序
- T：根据时间/累计时间进行排序

## 显示完整的程序命令
```
[~]# top -c
top - 14:19:47 up 419 days, 23:16,  2 users,  load average: 0.10, 0.04, 0.09
Tasks: 217 total,   1 running, 216 sleeping,   0 stopped,   0 zombie
%Cpu(s):  1.3 us,  0.6 sy,  0.0 ni, 98.1 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 32781768 total,  2991744 free,  7742820 used, 22047204 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 24486788 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                                                                                     
18162 chrony    20   0  976728 540512   5176 S   2.0  1.6   8734:11 puma 3.12.0 (tcp://localhost:9168) [gitlab-monitor]                                                                                                                         
 5257 root      10 -10  133448  15736   9520 S   1.0  0.0   1018:26 /usr/local/aegis/aegis_client/aegis_10_73/AliYunDun                                                                                                                         
17735 997       20   0  105124  16552   1608 S   1.0  0.1   1219:11 /opt/gitlab/embedded/bin/redis-server 127.0.0.1:0                                                                                                                           
18078 chrony    20   0 1973556 867296  12964 S   0.7  2.6   2702:14 sidekiq 5.2.5 gitlab-rails [0 of 25 busy]                                                                                                                                   
19608 root      20   0   44444  20336  11816 S   0.7  0.1   1003:26 gitlab-runner run --user=gitlab-runner --working-directory=/home/gitlab-runner                               
```

## 显示指定的进程信息
```
[~]# top -p 17735
top - 14:34:41 up 419 days, 23:31,  2 users,  load average: 0.00, 0.03, 0.08
Tasks:   1 total,   0 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  1.5 us,  0.4 sy,  0.0 ni, 98.1 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 32781768 total,  2988360 free,  7725668 used, 22067740 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 24504020 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                                                                                     
17735 997       20   0  105124  16552   1608 S   0.3  0.1   1219:16 redis-server 
```

## 显示指定的多个进程

```
[~]# top -p `pgrep java | tr "\\n" "," | sed 's/,$//'`
top - 14:36:39 up 105 days, 23:25,  1 user,  load average: 0.00, 0.02, 0.05
Tasks:   2 total,   0 running,   2 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.3 us,  0.3 sy,  0.0 ni, 99.5 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 32780448 total, 27889284 free,  1736588 used,  3154576 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 30614672 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                                                       
26649 root      20   0 4763552 719376  19032 S   0.3  2.2   3:00.31 java                                                                                                                                          
26704 root      20   0 4763400 760588  18900 S   0.3  2.3   3:03.21 java
```


# 参考资料

https://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/top.html