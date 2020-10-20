---
layout: post
title: linux命令-lsof
date: 2019-10-03
categories:
    - linux
comments: true
permalink: linux-command-lsof.html
---

# 1. lsof

lsof, LiSt Opened Files, 列出打开的文件, 听起来很简单的样子. 但想linux中很多其他工具一样,  lsof把这件简单的事情做到了炉火纯青. 因为Unix认为”一切皆文件”, 那么”打开的文件”就不仅仅是传统意义上打开的文件了,  还可以是网络/Unix域套接字, 匿名/具名管道, 共享库文件, 目录文件, 设备文件等等. 很多场景下,  查看进程或系统打开的文件会给调试带来极大的帮助。

lsof也是有着最多开关的Linux/Unix命令之一，它有许多选项支持使用-和+前缀。

```
~# lsof -h
lsof 4.86
 latest revision: ftp://lsof.itap.purdue.edu/pub/tools/unix/lsof/
 latest FAQ: ftp://lsof.itap.purdue.edu/pub/tools/unix/lsof/FAQ
 latest man page: ftp://lsof.itap.purdue.edu/pub/tools/unix/lsof/lsof_man
 usage: [-?abhKlnNoOPRtUvVX] [+|-c c] [+|-d s] [+D D] [+|-f[gG]] [+|-e s]
 [-F [f]] [-g [s]] [-i [i]] [+|-L [l]] [+m [m]] [+|-M] [-o [o]] [-p s]
[+|-r [t]] [-s [p:s]] [-S [t]] [-T [t]] [-u s] [+|-w] [-x [fl]] [--] [names]

```

# 2. 关键选项

理解一些关于lsof如何工作的关键性东西是很重要的。最重要的是，当你给它传递选项时，默认行为是对结果进行“或”运算。因此，如果你正是用-i来拉出一个端口列表，同时又用-p来拉出一个进程列表，那么默认情况下你会获得两者的结果。

下面的一些其它东西需要牢记：

- 默认 : 没有选项，lsof列出活跃进程的所有打开文件
- 组合 : 可以将选项组合到一起，如-abc，但要当心哪些选项需要参数
- -a : 结果进行“与”运算（而不是“或”）
- -l : 在输出显示用户ID而不是用户名
- -h : 获得帮助
- -t : 仅获取进程ID
- -U : 获取UNIX套接口地址
- -F : 格式化输出结果，用于其它命令。可以通过多种方式格式化，如-F pcfn（用于进程id、命令名、文件描述符、文件名，并以空终止）

# 3. 获取网络信息

## 3.1. 使用-i显示所有连接

语法: `lsof -i[46] [protocol][@hostname|hostaddr][:service|port]`

```
~# lsof -i
COMMAND     PID USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
IntelliJI  2003 root    3u  IPv4    18285      0t0  TCP *:1018 (LISTEN)
IntelliJI  2003 root    5u  IPv4   546501      0t0  TCP xxxx:1018->static.vnpt.vn:3866 (ESTABLISHED)
IntelliJI  2003 root    6u  IPv4   607981      0t0  TCP xxxx:1018->static.vnpt.vn:14328 (ESTABLISHED)
IntelliJI  2003 root    7u  IPv4   643684      0t0  TCP xxxx:1018->static.vnpt.vn:8299 (ESTABLISHED)
IntelliJI  2003 root    8u  IPv4   830937      0t0  TCP xxxx:1018->119.98.168.125:58913 (ESTABLISHED)
IntelliJI  2003 root    9u  IPv4   852131      0t0  TCP xxxx:1018->121.60.84.184:50177 (ESTABLISHED)
```

## 3.2. 获取IPV6流量

```
~# lsof -i 6
COMMAND     PID USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
ntpd      16045  ntp   17u  IPv6 13133615      0t0  UDP *:ntp
redis-sen 28757 root    6u  IPv6 10042992      0t0  TCP *:26379 (LISTEN)
redis-sen 28763 root    6u  IPv6 10042130      0t0  TCP *:26380 (LISTEN)
redis-sen 28769 root    6u  IPv6 10042153      0t0  TCP *:26381 (LISTEN)
redis-ser 31804 root    6u  IPv6 10054389      0t0  TCP *:6380 (LISTEN)
redis-ser 31821 root    6u  IPv6 10055495      0t0  TCP *:6381 (LISTEN)
redis-ser 31838 root    6u  IPv6 10055619      0t0  TCP *:6379 (LISTEN)
mysqld    32191 root   30u  IPv6 17244127      0t0  TCP *:33060 (LISTEN)

```

## 3.3. 仅显示UDP连接

```
~# lsof -i udp
COMMAND   PID USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
ntpd    16045  ntp   16u  IPv4 13133614      0t0  UDP *:ntp
ntpd    16045  ntp   17u  IPv6 13133615      0t0  UDP *:ntp
ntpd    16045  ntp   18u  IPv4 13133621      0t0  UDP localhost:ntp
ntpd    16045  ntp   19u  IPv4 13133622      0t0  UDP iZuf6fbha9mr8eysoj3mabZ:ntp
ntpd    16045  ntp   20u  IPv4 13133623      0t0  UDP 172.17.0.1:ntp
consul  18197 root    7u  IPv4 14297484      0t0  UDP localhost:8302
consul  18197 root    9u  IPv4 14297486      0t0  UDP localhost:8301
consul  18197 root   10u  IPv4 14298289      0t0  UDP localhost:8600
```

## 3.4. 显示与指定端口相关的网络信息

```
~# lsof -i :22
COMMAND   PID USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
sshd    20388 root    3u  IPv4 13152920      0t0  TCP *:ssh (LISTEN)
sshd    22018 root    3u  IPv4 41917229      0t0  TCP xxxx:ssh->xxxx:49428 (ESTABLISHED)
sshd    22033 root    3u  IPv4 41918512      0t0  TCP xxxx:ssh->xxxx:49429 (ESTABLISHED)
```

## 3.5. 显示指定到指定主机的连接

```
~# lsof -i @xxxx
COMMAND   PID USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
sshd    22018 root    3u  IPv4 41917229      0t0  TCP xxxx:ssh->xxxx:49428 (ESTABLISHED)
sshd    22033 root    3u  IPv4 41918512      0t0  TCP xxxx:ssh->xxxx:49429 (ESTABLISHED)
```

## 3.6. 显示基于主机与端口的连接

```
~# lsof -i @xxxx:49428
COMMAND   PID USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
sshd    22018 root    3u  IPv4 41917229      0t0  TCP xxxx:ssh->xxxx:49428 (ESTABLISHED)
```

## 3.7. 找出监听端口

```
~# lsof -i -s TCP:listen
COMMAND     PID USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
IntelliJI  2003 root    3u  IPv4    18285      0t0  TCP *:1018 (LISTEN)
java      17926 root   16u  IPv4 13891446      0t0  TCP *:52974 (LISTEN)
java      17926 root   21u  IPv4 13891452      0t0  TCP localhost:33756 (LISTEN)
...
```

也可以使用`lsof -i | grep LISTEN`

## 3.8. 找出已建立的连接

```
~# lsof -i -s TCP:ESTABLISHED
```

# 4. 用户信息

你也可以获取各种用户的信息，以及它们在系统上正干着的事情，包括它们的网络活动、对文件的操作等。

## 4.1. 显示指定用户打开了什么

```
~# lsof -u root
```

## 4.2. 显示除指定用户以外的其它所有用户所做的事情

```
~# lsof -u ^root
```

## 4.3. 杀死指定用户所做的一切事情

```
~# kill  -9  `lsof -t -u daniel`
```

# 5. 命令和进程

可以查看指定程序或进程由什么启动，这通常会很有用，而你可以使用lsof通过名称或进程ID过滤来完成这个任务

## 5.1. 查看指定的命令正在使用的文件和网络连接

```
~# lsof -c java
```

> 可以指定多个

## 5.2.  查看指定进程ID已打开的内容

```
~# lsof -p 25951
```

## 5.3. 只返回PID

```
~# lsof -c java -t
17926
25373
25411
25941
```

# 6. 文件和目录

通过查看指定文件或目录，你可以看到系统上所有正与其交互的资源——包括用户、进程等。

## 6.1. 显示与指定目录交互的所有一切

```
~# lsof  /var/log/messages/
```

## 6.2. 显示与指定文件交互的所有一切

```
~# lsof  /home/daniel/firewall_whitelist.txt
```

# 7. 其他用法

`lsof -a``: 上述功能性选项可以组合使用, 但默认采用OR逻辑列出, -a选项令lsof使用AND逻辑;

`lsof +D .` : 递归地列出当前目录中被打开的文件, 当然也可以lsof | grep `pwd`;

`lsof -r [seconds]`: -r选项可以让lsof以一定的时间间隔连续执行, 在监视文件/进程时会非常实用.

`lsof  -i @fw.google.com:2150=2180`: 显示某个端口范围的打开的连接

`lsof  +L1`: 显示所有打开的链接数小于1的文件，这通常（当不总是）表示某个攻击者正尝试通过删除文件入口来隐藏文件内容

` lsof -a  -u root -i @1.1.1.1`: 显示root连接到1.1.1.1所做的一切

# 8. 参考资料

https://www.jianshu.com/p/a3aa6b01b2e1