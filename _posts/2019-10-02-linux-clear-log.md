---
layout: post
title: linux系统清空日志/var/log/messages
date: 2019-10-02
categories:
    - linux
comments: true
permalink: linux-clear-log.html
---

# 1. 系统日志`/var/log/messages`

`/var/log/messages` 存放的是系统的日志信息，它记录了各种事件，基本上什么应用都能往里写日志，在做故障诊断时可以首先查看该文件内容

```
# tail /var/log/messages 
Dec 18 23:27:07 localhost kernel: [33100840.027421] possible SYN flooding on port 3873. Sending cookies.
Dec 18 23:30:11 localhost sz[31090]: [root] 1.txt/ZMODEM: 8144 Bytes, 16520 BPS
Dec 19 16:29:29 localhost kernel: [33162181.801121] possible SYN flooding on port 3873. Sending cookies.  
Dec 19 17:15:05 localhost sz[16756]: [root] ConfigActivity.xml/ZMODEM: 653205 Bytes, 393989 BPS
Dec 19 17:38:55 localhost sz[17617]: [root] ConfigActivity.xml/ZMODEM: 287531 Bytes, 337878 BPS
```

可以分为几个字段来描述这些事件信息：

　　1. 事件的日期和时间
　　2. 事件的来源主机
　　3. 产生这个事件的程序[进程号] 
　　4. 实际的日志信息

如：`Dec 19 17:39:06 localhost sz[17833]: [root] ConfigActivity.xml/ZMODEM: 287531 Bytes, 343933 BPS`

　　1. 产生这个事件的时间是：Dec 19 17:39:06
　　2. 事件的来源主机为：localhost
　　3. 产生这个事件的程序和进程号为：sz[17833]
　　4. 这个事件实际的日志信息为：[root] ConfigActivity.xml/ZMODEM: 287531 Bytes, 343933 BPS

# 2. 清理`/var/log/messages`

Linux新手容易犯的一个错误是把日志文件给直接删除，而不是删除日志文件的内容。但是直接删除日志文件往往导致新产生的日志记录无法被写入到日志文件中（因为它已经被删除了），而仅仅重新新建（touch）同样名字的文件是解决不了问题的。

首先，在以root用户执行如下lsof命令，查询打开`/var/log/messages`文件的进程的进程ID(PID)：

```
lsof | grep messages 
```

得到输出：

```
COMMAND   PID USER  FD  TYPE DEVICE  SIZE/OFF NODE NAME 

rsyslogd  1455 root   4w  REG   8,6 1299113404 2686 messages 

abrt-dump 1932 root   4r  REG   8,6 1299113404 2686 messages 
```

结束生成messages的进程：

```
kill -9 1455 

kill -9 1932 
```

清空日志并重启：

```
cat /dev/null > /var/log/messages 

reboot 
```

# 3. 参考资料

https://blog.csdn.net/qq_38791687/article/details/90370910