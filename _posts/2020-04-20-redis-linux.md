---
layout: post
title: redis在Linux系统的配置优化 
date: 2020-04-20
categories:
    - redis
comments: true
permalink: redis-linux.html
---

> https://mp.weixin.qq.com/s/eN8qQn9HjeI1BV-MMFcWiw

# 1. 内存分配控制

Redis在启动时可能会出现这样的日志：

```
# WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the 
command 'sysctl vm.overcommit_memory=1' for this to take effect.
```

Linux操作系统对大部分申请内存的请求都回复yes，以便能运行更多的程序。因为申请内存后，并不会马上使用内存，这种技术叫做overcommit。如果Redis在启动时有上面的日志，说明vm.overcommit_memory=0，Redis提示把它设置为1。

vm.overcommit_memory用来设置内存分配策略，它有三个可选值，

- **0**	表示内核将检查是否有足够的可用内存。如果有足够的可用内存，内存申请通过，否则内存申请失败，并把错误返回给应用进程
- **1**	表示内核允许超量使用内存直到用完为止
- **2**	表示内核决不过量的("never overcommit")使用内存，即系统整个内存地址空间不能超过swap+50%的RAM值，50%是overcommit_ratio默认值，此参数同样支持修改

日志中的Background  save代表的是bgsave和bgrewriteaof，如果当前可用内存不足，操作系统应该如何处理fork。如果vm.overcommit_memory=0，代表如果没有可用内存，就申请内存失败，对应到Redis就是fork执行失败，在Redis的日志会出现：

```
Cannot allocate memory 
```

Redis建议把这个值设置为1，是为了让fork能够在低内存下也执行成功。

```
# 查看
# cat /proc/sys/vm/overcommit_memory
0
# 设置
echo "vm.overcommit_memory=1" >> /etc/sysctl.conf
sysctl vm.overcommit_memory=1
```

# 2. swappiness

swap对于操作系统来比较重要，当物理内存不足时，可以swap  out一部分内存页，以解燃眉之急。但世界上没有免费午餐，swap空间由硬盘提供，对于需要高并发、高吞吐的应用来说，磁盘IO通常会成为系统瓶颈。在Linux中，并不是要等到所有物理内存都使用完才会使用到swap，系统参数swppiness会决定操作系统使用swap的倾向程度。swappiness的取值范围是0~100，swappiness的值越大，说明操作系统可能使用swap的概率越高，swappiness值越低，表示操作系统更加倾向于使用物理内存。swap的默认值是60，了解这个值的含义后，有利于Redis的性能优化。

- **0**	Linux3.5以及以上：宁愿OOM killer也不用swap，Linux3.4以及更早：宁愿swap也不要OOM killer
- **1**	Linux3.5以及以上：宁愿swap也不要OOM killer
- **60**	默认值
- **100**	操作系统会主动地使用swap

swappiness设置方法如下：

```
echo {bestvalue} > /proc/sys/vm/swappiness
```

但是上述方法在系统重启后就会失效，为了让配置在重启Linux操作系统后立即生效，只需要在/etc/sysctl.conf追加 vm.swappiness={bestvalue}即可。

```
echo vm.swappiness={bestvalue} >> /etc/sysctl.conf
```

如果Linux>3.5，vm.swapniess=1，否则vm.swapniess=0，从而实现如下两个目标：

- 物理内存充足时候，使Redis足够快。
- 物理内存不足时候，避免Redis死掉(如果当前Redis为高可用，死掉比阻塞更好)。

# 3. Transparent Huge Pages

Redis在启动时可能会看到如下日志：

```
WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
```

从提示看Redis建议修改Transparent Huge Pages (THP)的相关配置，Linux kernel在2.6.38内核增加了Transparent Huge Pages  (THP)特性  ，支持大内存页(2MB)分配，默认开启。当开启时可以降低fork子进程的速度，但fork之后，每个内存页从原来4KB变为2MB，会大幅增加重写期间父进程内存消耗。同时每次写命令引起的复制内存页单位放大了512倍，会拖慢写操作的执行时间，导致大量写操作慢查询。例如简单的incr命令也会出现在慢查询中。因此Redis日志中建议将此特性进行禁用，禁用方法如下：

```
echo never >  /sys/kernel/mm/transparent_hugepage/enabled
```

而且为了使机器重启后THP配置依然生效，可以在/etc/rc.local中追加echo never > /sys/kernel/mm/transparent_hugepage/enabled。

在设置THP配置时需要注意：有些Linux的发行版本没有将THP放到/sys/kernel/mm/transparent_hugepage/enabled中，例如Red Hat  6以上的THP配置放到/sys/kernel/mm/redhat_transparent_hugepage/enabled中。而Redis源码中检查THP时，把THP位置写死。

```
FILE *fp = fopen("/sys/kernel/mm/transparent_hugepage/enabled","r");
if (!fp) return 0;   
```

所以在发行版中，虽然没有THP的日志提示，但是依然存在THP所带来的问题。

```
echo never >  /sys/kernel/mm/redhat_transparent_hugepage/enabled
```

# 4. OOM killer

OOM killer会在可用内存不足时选择性的杀掉用户进程，它的运行规则是怎样的，会选择哪些用户进程“下手”呢？OOM  killer进程会为每个用户进程设置一个权值，这个权值越高，被“下手”的概率就越高，反之概率越低。每个进程的权值存放在/proc/{progress_id}/oom_score中，这个值是受/proc/{progress_id}/oom_adj的控制，oom_adj在不同的Linux版本的最小值不同。当oom_adj设置为最小值时，该进程将不会被OOM killer杀掉，设置方法如下。

```
echo {value} > /proc/${process_id}/oom_adj
```

对于Redis所在的服务器来说，可以将所有Redis的oom_adj设置为最低值或者稍小的值，降低被OOM killer杀掉的概率。

```
for redis_pid in $(pgrep -f "redis-server")
do
  echo -17 > /proc/${redis_pid}/oom_adj
done
```

- oom_adj参数只能起到辅助作用，合理的规划内存更为重要。
- 通常在高可用情况下，被杀掉比僵死更好，因此不要过多依赖oom_adj配置

# 5. ulimit

在Linux中，可以通过ulimit查看和设置系统的当前用户进程的资源数。其中ulimit -a命令包含的open files参数，是单个用户同时打开的最大文件个数。

```
# ulimit –a
…
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
…
```

Redis允许同时有多个客户端通过网络进行连接，可以通过配置maxclients来限制最大客户端连接数。对Linux操作系统来说这些网络连接都是文件句柄。假设当前open files是4096，那么启动Redis时会看到如下日志。

```
# You requested maxclients of 10000 requiring at least 10032 max file descriptors.
# Redis can’t set maximum open files to 10032 because of OS error: Operation not permitted.
# Current maximum open files is 4096. Maxclients has been reduced to 4064 to compensate for low ulimit. If you need higher maxclients increase ‘ulimit –n’.
```

上面的日志解释如下：

- 第一行：Redis建议把open  files至少设置成10032，那么这个10032是如何来的呢？因为maxclients的默认是10000，这些是用来处理客户端连接的，除此之外，Redis内部会使用最多32个文件描述符，所以这里的10032 = 10000 + 32。
- 第二行：Redis不能将open files设置成10032，因为它没有权限设置。
- 第三行：当前系统的open files是4096，所以maxclients被设置成4096-32=4064个，如果你想设置更高的maxclients，请使用ulimit  -n来设置。从上面的三行日志分析可以看出open files的限制优先级比maxclients大。open files的设置方法如下：

```
ulimit –Sn {max-open-files}
```

# 6. TCP backlog

Redis默认的tcp-backlog为511，可以通过修改配置tcp-backlog进行调整，如果Linux的tcp-backlog小于Redis设置的tcp-backlog，那么在Redis启动时会看到如下日志：

```
# WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
```

查看方法：

```
cat /proc/sys/net/core/somaxconn
128
```

修改方法：.

```
echo 511 > /proc/sys/net/core/somaxconn
```