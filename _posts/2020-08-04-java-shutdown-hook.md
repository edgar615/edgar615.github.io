---
layout: post
title: Java ShutdownHook
date: 2020-05-21
categories:
    - java
comments: true
permalink: java-shutdown-hook.html
---

# 1. 信号

在Linux中，信号是进程间通讯的一种方式，它采用的是异步机制。当信号发送到某个进程中时，操作系统会中断该进程的正常流程，并进入相应的信号处理函数执行操作，完成后再回到中断的地方继续执行。

每个信号都有自己的响应动作，当接收到信号时，进程会根据信号的响应动作执行相应的操作，信号的响应动作有以下几种：

- 中止进程(Term)
- 忽略信号(Ign)
- 中止进程并保存内存信息(Core)
- 停止进程(Stop)
- 继续运行进程(Cont)

停机信号

在linux上，我们停机主要是使用 kill 的方式。我们平时经常使用的两种：

- kill -15 <pid> 结束程序(可以被捕获、阻塞或忽略) kill的默认信号，可以理解为发送一个通知，告知应用主动关闭
- kill -9 <pid> 无条件结束程序(不能被捕获、阻塞或忽略)，可以理解为操作系统从内核级别强行杀死某个进程

> 在 Linux 中 kill 指令负责杀死进程，其后可以紧跟一个数字，代表 信号编号 (Signal)，执行 kill -l 指令，可以一览所有的信号编号。
>
> $ kill -l
>  1) SIGHUP	 2) SIGINT	 3) SIGQUIT	 4) SIGILL	 5) SIGTRAP
>  6) SIGABRT	 7) SIGBUS	 8) SIGFPE	 9) SIGKILL	10) SIGUSR1
> 11) SIGSEGV	12) SIGUSR2	13) SIGPIPE	14) SIGALRM	15) SIGTERM
> 16) SIGSTKFLT	17) SIGCHLD	18) SIGCONT	19) SIGSTOP	20) SIGTSTP
> 21) SIGTTIN	22) SIGTTOU	23) SIGURG	24) SIGXCPU	25) SIGXFSZ
> 26) SIGVTALRM	27) SIGPROF	28) SIGWINCH	29) SIGIO	30) SIGPWR
> 31) SIGSYS	34) SIGRTMIN	35) SIGRTMIN+1	36) SIGRTMIN+2	37) SIGRTMIN+3
> 38) SIGRTMIN+4	39) SIGRTMIN+5	40) SIGRTMIN+6	41) SIGRTMIN+7	42) SIGRTMIN+8
> 43) SIGRTMIN+9	44) SIGRTMIN+10	45) SIGRTMIN+11	46) SIGRTMIN+12	47) SIGRTMIN+13
> 48) SIGRTMIN+14	49) SIGRTMIN+15	50) SIGRTMAX-14	51) SIGRTMAX-13	52) SIGRTMAX-12
> 53) SIGRTMAX-11	54) SIGRTMAX-10	55) SIGRTMAX-9	56) SIGRTMAX-8	57) SIGRTMAX-7
> 58) SIGRTMAX-6	59) SIGRTMAX-5	60) SIGRTMAX-4	61) SIGRTMAX-3	62) SIGRTMAX-2
> 63) SIGRTMAX-1	64) SIGRTMAX

# 2. ShutdownHook
Java 语言提供一种 ShutdownHook（钩子）进制，当 JVM 接受到系统的关闭通知之后，调用 ShutdownHook 内的方法，用以完成清理操作，从而平滑的退出应用。

```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
	System.out.println("关闭应用，释放资源");
}));
```
