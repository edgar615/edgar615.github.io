---
layout: post
title: Linux监控工具-dstat
date: 2019-10-30
categories:
    - linux
comments: true
permalink: linux-monitor-dstat.html
---

dstat 是一个可以取代vmstat，iostat，netstat和ifstat这些命令的多功能产品。dstat克服了这些命令的局限并增加了一些另外的功能，增加了监控项，也变得更灵活了。dstat可以很方便监控系统运行状况并用于基准测试和排除故障。

dstat可以让你实时地看到所有系统资源，例如，你能够通过统计IDE控制器当前状态来比较磁盘利用率，或者直接通过网络带宽数值来比较磁盘的吞吐率（在相同的时间间隔内）。

dstat将以列表的形式为你提供选项信息并清晰地告诉你是在何种幅度和单位显示输出。这样更好地避免了信息混乱和误报。更重要的是，它可以让你更容易编写插件来收集你想要的数据信息，以从未有过的方式进行扩展。

Dstat的默认输出是专门为人们实时查看而设计的，不过你也可以将详细信息通过CSV输出到一个文件，并导入到Gnumeric或者Excel生成表格中。

# 语法

	[root@ihorn-dev ~]# dstat -h
	Usage: dstat [-afv] [options..] [delay [count]]
	Versatile tool for generating system resource statistics
	
	Dstat options:
	  -c, --cpu              enable cpu stats
	     -C 0,3,total           include cpu0, cpu3 and total
	  -d, --disk             enable disk stats
	     -D total,hda           include hda and total
	  -g, --page             enable page stats
	  -i, --int              enable interrupt stats
	     -I 5,eth2              include int5 and interrupt used by eth2
	  -l, --load             enable load stats
	  -m, --mem              enable memory stats
	  -n, --net              enable network stats
	     -N eth1,total          include eth1 and total
	  -p, --proc             enable process stats
	  -r, --io               enable io stats (I/O requests completed)
	  -s, --swap             enable swap stats
	     -S swap1,total         include swap1 and total
	  -t, --time             enable time/date output
	  -T, --epoch            enable time counter (seconds since epoch)
	  -y, --sys              enable system stats
	
	  --aio                  enable aio stats
	  --fs, --filesystem     enable fs stats
	  --ipc                  enable ipc stats
	  --lock                 enable lock stats
	  --raw                  enable raw stats
	  --socket               enable socket stats
	  --tcp                  enable tcp stats
	  --udp                  enable udp stats
	  --unix                 enable unix stats
	  --vm                   enable vm stats
	
	  --plugin-name          enable plugins by plugin name (see manual)
	  --list                 list all available plugins
	
	  -a, --all              equals -cdngy (default)
	  -f, --full             automatically expand -C, -D, -I, -N and -S lists
	  -v, --vmstat           equals -pmgdsc -D total
	
	  --bits                 force bits for values expressed in bytes
	  --float                force float values on screen
	  --integer              force integer values on screen
	
	  --bw, --blackonwhite   change colors for white background terminal
	  --nocolor              disable colors (implies --noupdate)
	  --noheaders            disable repetitive headers
	  --noupdate             disable intermediate updates
	  --output file          write CSV output to file
	  --profile              show profiling statistics when exiting dstat
	
	delay is the delay in seconds between each update (default: 1)
	count is the number of updates to display before exiting (default: unlimited)

参数

    -c：显示CPU系统占用，用户占用，空闲，等待，中断，软件中断等信息。
    -C：当有多个CPU时候，此参数可按需分别显示cpu状态，例：-C 0,1 是显示cpu0和cpu1的信息。 -d：显示磁盘读写数据大小。 
    -D hda,total：include hda and total。 
    -n：显示网络状态。 
    -N eth1,total：有多块网卡时，指定要显示的网卡。 
    -l：显示系统负载情况。 
    -m：显示内存使用情况。 
    -g：显示页面使用情况。 
    -p：显示进程状态。 
    -s：显示交换分区使用情况。 
    -S：类似D/N。 
    -r：I/O请求情况。 
    -y：系统状态。 
    --ipc：显示ipc消息队列，信号等信息。 
    --socket：用来显示tcp udp端口状态。 
    -a：此为默认选项，等同于-cdngy。 
    -v：等同于 -pmgdsc -D total。 
    --output 文件：此选项也比较有用，可以把状态信息以csv的格式重定向到指定的文件中，以便日后查看。例：dstat --output /root/dstat.csv & 此时让程序默默的在后台运行并把结果输出到/root/dstat.csv文件中。

# 示例

	[root@ihorn-dev ~]# dstat
	You did not select any stats, using -cdngy by default.
	----total-cpu-usage---- -dsk/total- -net/total- ---paging-- ---system--
	usr sys idl wai hiq siq| read  writ| recv  send|  in   out | int   csw 
	  7   3  88   0   0   1| 497B  168k|   0     0 |   0     0 |1823    11k
	  3   1  95   0   0   0|   0   316k|  64k   11k|   0     0 |2846  4946 
	  2   1  97   0   0   0|   0    20k|  24k  963B|   0     0 |2469  4495 
	  3   1  96   0   0   0|   0  4096B|  45k 4914B|   0     0 |2643  4689 
	  3   1  96   0   0   0|4096B   96k|  31k 6194B|   0     0 |2765  4758 
	  3   1  96   0   0   0|   0    20k|  52k 9956B|   0     0 |2703  4706 
	  3   1  95   0   0   0|   0   328k|  61k   16k|   0     0 |2803  4899 
	 10   2  86   3   0   0|   0  1264k|  87k   22k|   0     0 |4051  6075 

输出项说明

**CPU状态**：CPU的使用率。这项报告更有趣的部分是显示了用户，系统和空闲部分，这更好地分析了CPU当前的使用状况。如果你看到"wait"一栏中，CPU的状态是一个高使用率值，那说明系统存在一些其它问题。当CPU的状态处在"waits"时，那是因为它正在等待I/O设备（例如内存，磁盘或者网络）的响应而且还没有收到。

**磁盘统计**：磁盘的读写操作，这一栏显示磁盘的读、写总数。

**网络统计**：网络设备发送和接受的数据，这一栏显示的网络收、发数据总数。

**分页统计**：系统的分页活动。分页指的是一种内存管理技术用于查找系统场景，一个较大的分页表明系统正在使用大量的交换空间，或者说内存非常分散，大多数情况下你都希望看到page in（换入）和page out（换出）的值是0 0。

**系统统计**：这一项显示的是中断（int）和上下文切换（csw）。这项统计仅在有比较基线时才有意义。这一栏中较高的统计值通常表示大量的进程造成拥塞，需要对CPU进行关注。你的服务器一般情况下都会运行运行一些程序，所以这项总是显示一些数值。



# 插件
dstat附带了一些插件很大程度地扩展了它的功能。你可以通过查看/usr/share/dstat目录来查看它们的一些使用方法，常用的有这些：

    -–disk-util ：显示某一时间磁盘的忙碌状况
    -–freespace ：显示当前磁盘空间使用率
    -–proc-count ：显示正在运行的程序数量
    -–top-bio ：指出块I/O最大的进程
    -–top-cpu ：图形化显示CPU占用最大的进程
    -–top-io ：显示正常I/O最大的进程
    -–top-mem ：显示占用最多内存的进程

# 查看全部内存都有谁在占用

	[root@ihorn-dev ~]# dstat -g -l -m -s --top-mem
	---paging-- ---load-avg--- ------memory-usage----- ----swap--- --most-expensive-
	  in   out | 1m   5m  15m | used  buff  cach  free| used  free|  memory process 
	   0     0 |0.21 0.23 0.23|7141M  204M  888M 7655M|   0     0 |node /alidat4746G
	   0     0 |0.21 0.23 0.23|7141M  204M  888M 7655M|   0     0 |node /alidat4746G
	   0     0 |0.21 0.23 0.23|7143M  204M  888M 7654M|   0     0 |node /alidat4746G
	   0     0 |0.19 0.23 0.23|7141M  204M  888M 7655M|   0     0 |node /alidat4746G
	   0     0 |0.19 0.23 0.23|7143M  204M  888M 7653M|   0     0 |node /alidat4746G
	   0     0 |0.19 0.23 0.23|7140M  204M  888M 7655M|   0     0 |node /alidat4746G
	   0     0 |0.19 0.23 0.23|7140M  204M  888M 7656M|   0     0 |node /alidat4746G

# 显示一些关于CPU资源损耗的数据：

	[root@ihorn-dev ~]# dstat -c -y -l --proc-count --top-cpu
	----total-cpu-usage---- ---system-- ---load-avg--- proc -most-expensive-
	usr sys idl wai hiq siq| int   csw | 1m   5m  15m |tota|  cpu process   
	  7   3  88   0   0   1|1823    11k|0.10 0.18 0.21| 193|java         6.7
	  3   1  96   0   0   0|2785  4893 |0.10 0.18 0.21| 193|node /alidata1.2
	  3   1  96   0   0   0|2822  4930 |0.10 0.18 0.21| 193|node /alidata1.2

# 如何输出一个csv文件

	dstat –output /tmp/sampleoutput.csv -cdn
	
# 参考资料

https://linux.cn/article-3215-1.html