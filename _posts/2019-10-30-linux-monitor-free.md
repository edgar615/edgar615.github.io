---
layout: post
title: Linux监控工具-free
date: 2019-10-30
categories:
    - linux
comments: true
permalink: linux-monitor-free.html
---

Linux “free”命令可以给出类Linux/Unix操作系统中物理内存和交换内存的总使用量、可用量及内核使用的缓冲区情况。


# 语法

	[root@ihorn-dev ~]# free --help
	
	Usage:
	 free [options]
	
	Options:
	 -b, --bytes         show output in bytes
	 -k, --kilo          show output in kilobytes
	 -m, --mega          show output in megabytes
	 -g, --giga          show output in gigabytes
	     --tera          show output in terabytes
	 -h, --human         show human-readable output
	     --si            use powers of 1000 not 1024
	 -l, --lohi          show detailed low and high memory statistics
	 -o, --old           use old format (without -/+buffers/cache line)
	 -t, --total         show total for RAM + swap
	 -s N, --seconds N   repeat printing every N seconds
	 -c N, --count N     repeat printing N times, then exit
	
	     --help     display this help and exit
	 -V, --version  output version information and exit
	
	For more details see free(1).

参数说明

	-b：以Byte为单位显示内存使用情况； 
	-k：以KB为单位显示内存使用情况； 
	-m：以MB为单位显示内存使用情况； 
	-g：以GB为单位显示内存使用情况；
	-o：不显示缓冲区调节列； 
	-s<间隔秒数>：持续观察内存使用状况； 
	-t：显示内存总和列； 
	-V：显示版本信息
	-l：显示具体的高和低内存的使用统计情况


示例

	[root@ihorn-dev ~]# free 
	             total       used       free     shared    buffers     cached
	Mem:      16269468   14629084    1640384     213968     387248    6444076
	-/+ buffers/cache:    7797760    8471708
	Swap:            0          0          0

显示内容说明


    total：物理内存大小，就是机器实际的内存
    used：已使用的内存大小，这个值包括了 cached 和 应用程序实际使用的内存
    free：未被使用的内存大小
    shared：共享内存大小，是进程间通信的一种方式
    buffers：被缓冲区占用的内存大小，后面会详细介绍
    cached：被缓存占用的内存大小，后面会详细介绍

真正被使用的内存real free memory ＝ used - buffers - cached；
真正可用的内存real free memory = total - real free memory

下面一行，代表应用程序实际使用的内存：

    前一个值表示 - buffers/cached，即 used - buffers/cached，表示应用程序实际使用的内存
    后一个值表示 + buffers/cached，即 free + buffers/cached，表示理论上都可以被使用的内存

第三行表示 swap 的使用情况：总量、使用的和未使用的

# Cache
cache 就是缓存的意思。当系统读文件的时候，都是把数据从硬盘读到内存里，因为硬盘比内存慢很多，所以这个过程会很耗时。为了提高效率，linux 会把读进来的文件在内存中缓存下来（因为读取相近部分的内容是程序很常见的情况），即使程序结束，cache 也不会被自动释放。所以呢，如果有程序进行大量的读文件操作，你会发现内存使用率就上去了。

不过也不用担心，如果其他程序使用要使用内存的时候，linux 也会把这些没人使用的 cache 释放掉，给其他运行的程序使用。当然你也可以手动去释放掉这部分内存：

	echo 1 > /proc/sys/vm/drop_caches

# Buffer

buffer 的意思和 cache 相近，不过稍有区别。考虑内存写文件到硬盘的过程，因为硬盘太慢了，如果内存要等待数据写完之后才继续后面的操作，实在是效率很低的事情，也会影响程序的运行速度。所以就有了 buffer，写到硬盘的数据会放到 buffer 里面，内存很快把数据写到 buffer，可以继续其他的工作，而硬盘可以在后台慢慢读出 buffer 中的数据，保存起来。这样就提高了读写的效率！

讲一个大家会经常遇到的例子，当我们把电脑里中的文件拷贝到 U 盘的时候，如果文件特别大，大家会遇到这种情况：明明看到文件已经拷贝完了，但系统还是会提示 U 盘正在使用中。这就是 buffer 的原因，拷贝程序把东西放到 buffer 之后，但是 U 盘还没有写完。

同样的，可以手动来 flush buffer 的内容，使用的命令是 sync。

# swap

swap 是实现虚拟内存的重要概念。如果系统的负载太大，内存被用完，可能会出现严重的问题。swap 就是把硬盘上一部分空间当做内存使用，正在运行程序会使用物理内存，把没有正在使用的内存放到硬盘，这叫做 swap out；而把硬盘 swap 部分的内存重新放到物理内存中，叫做 swap in。

swap 可以再逻辑上扩大内存空间，但是会造成系统变慢，因为硬盘读写速度很慢。linux 系统比较智能，会把那些不怎么频繁使用的内存放到 swap。

# 参考资料

http://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/free.html

https://linux.cn/article-4755-1.html
