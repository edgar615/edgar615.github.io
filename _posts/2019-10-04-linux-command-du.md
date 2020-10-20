---
layout: post
title: linux命令-du
date: 2019-10-04
categories:
    - linux
comments: true
permalink: linux-command-du.html
---

**du命令**也是查看使用空间的，但是与df命令不同的是Linux du命令是对文件和目录磁盘使用的空间的查看，还是和df命令有一些区别的。

```
-a或-all 显示目录中个别文件的大小。
-b或-bytes 显示目录或文件大小时，以byte为单位。
-c或--total 除了显示个别目录或文件的大小外，同时也显示所有目录或文件的总和。
-k或--kilobytes 以KB(1024bytes)为单位输出。
-m或--megabytes 以MB为单位输出。
-s或--summarize 仅显示总计，只列出最后加总的值。
-h或--human-readable 以K，M，G为单位，提高信息的可读性。
-x或--one-file-xystem 以一开始处理时的文件系统为准，若遇上其它不同的文件系统目录则略过。
-L<符号链接>或--dereference<符号链接> 显示选项中所指定符号链接的源文件大小。
-S或--separate-dirs 显示个别目录的大小时，并不含其子目录的大小。
-X<文件>或--exclude-from=<文件> 在<文件>指定目录或文件。
--exclude=<目录或文件> 略过指定的目录或文件。
-D或--dereference-args 显示指定符号链接的源文件大小。
-H或--si 与-h参数相同，但是K，M，G是以1000为换算单位。
-l或--count-links 重复计算硬件链接的文件。
```

- **显示目录或者文件所占空间** 

```
# du 
44      ./xml
60      ./mem
8       ./thread
188     ./glog/glog-0.3.3/src/glog
32      ./glog/glog-0.3.3/src/base
96      ./glog/glog-0.3.3/src/windows/glog
124     ./glog/glog-0.3.3/src/windows
780     ./glog/glog-0.3.3/src
8       ./glog/glog-0.3.3/vsprojects/logging_unittest
8       ./glog/glog-0.3.3/vsprojects/logging_unittest_static
12      ./glog/glog-0.3.3/vsprojects/libglog_static
12      ./glog/glog-0.3.3/vsprojects/libglog
44      ./glog/glog-0.3.3/vsprojects
376     ./glog/glog-0.3.3/m4
248     ./glog/glog-0.3.3/.deps
32      ./glog/glog-0.3.3/doc
7180    ./glog/glog-0.3.3/.libs
8       ./glog/glog-0.3.3/packages/rpm
48      ./glog/glog-0.3.3/packages/deb
68      ./glog/glog-0.3.3/packages
17828   ./glog/glog-0.3.3
17836   ./glog
60      ./sort
43687
```

只显示当前目录下面的子目录的目录大小和当前目录的总的大小，最下面的43687为当前目录的总大小

- **显示指定文件所占空间**

```
# du a.out 
16      a.out
```

- **查看指定目录的所占空间**

```
# du gdb
80      gdb
```

- **显示多个文件所占空间**

```
# du a.out main.cpp 
16      a.out
4       main.cpp
```

- **只显示总和的大小**

```
# du -s
383124  .
# du -s redis
39784   redis
```

- **方便阅读的格式显示**

```
# du -h log
16K     log/boost_log/log
6.8M    log/boost_log
1.1M    log/logouts/test
1.1M    log/logouts
7.9M    log
```

- **文件和目录都显示**

```
# du -ah log
4.0K    log/main.cpp
4.0K    log/Log.h
12K     log/boost_log/log/sign_2016-06-21_20.000.log
16K     log/boost_log/log
4.0K    log/boost_log/main.cpp
8.0K    log/boost_log/mainEx.cc
2.5M    log/boost_log/a.out
3.9M    log/boost_log/Logger.o
4.0K    log/boost_log/Logger.h
448K    log/boost_log/main.o
4.0K    log/boost_log/Makefile
4.0K    log/boost_log/Logger.cpp
6.8M    log/boost_log
8.0K    log/Log.cpp
4.0K    log/logouts/test/x.txt
1.1M    log/logouts/test/20160614.log
1.1M    log/logouts/test
1.1M    log/logouts
7.9M    log
```

- **显示几个文件或目录各自占用磁盘空间的大小，还统计它们的总和**

```
# du -c md5 log 
7316    md5
16      log/boost_log/log
6928    log/boost_log
1116    log/logouts/test
1120    log/logouts
8068    log
15384   total
```

说明：

加上-c选项后，du不仅显示两个目录各自占用磁盘空间的大小，还在最后一行统计它们的总和。

- **按照空间大小排序**

```
# du | sort -nr | more
8068    .
6928    ./boost_log
1120    ./logouts
1116    ./logouts/test
16      ./boost_log/log
```

- **输出当前目录下各个子目录所使用的空间**

```
# du -h  --max-depth=1
7.9M    ./log
44K     ./xml
60K     ./mem
8.0K    ./thread
18M     ./glog
60K     ./sort
296M    ./python
39M     ./redis
80K     ./gdb
44K     ./minmaxheap
6.7M    ./async
7.2M    ./md5
28K     ./move
375M    .
```

- **找出在一个path下的最大文件**

```
du -sh [dirname|filename]
```
- **当前目录大小**

```
du -sh .
```
- **当前目录下文件或目录的大小**

```
du -sh *
```
- **显示前十个占用空间最大的文件或目录**

```
du -s * | sort -nr | head
```
说明：
- -h已易读的格式显示指定目录或文件的大小
- -s选项指定对于目录不详细显示每个子目录或文件的大小

`du -s * | sort -nr | head` 选出排在前面的10个，

`du -s * | sort -nr | tail` 选出排在后面的10个。