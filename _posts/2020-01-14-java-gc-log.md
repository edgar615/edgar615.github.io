---
layout: post
title: Java GC日志
date: 2020-01-14
categories:
    - JVM
comments: true
permalink: java-gc-log.html
---



可以通过在java命令种加入参数来指定对应的gc类型，打印gc日志信息并输出至文件等策略。

- -XX:+PrintGC 输出GC日志
- -XX:+PrintGCDetails 输出GC的详细日志
- -XX:+PrintGCTimeStamps 输出GC的时间戳（以基准时间的形式）
- -XX:+PrintGCDateStamps 输出GC的时间戳（以日期的形式，如 2013-05-04T21:53:59.234+0800）
- -XX:+PrintHeapAtGC 在进行GC的前后打印出堆的信息
- -Xloggc:../logs/gc.log 日志文件的输出路径



# 参考资料

https://www.jianshu.com/p/fd1d4f21733a