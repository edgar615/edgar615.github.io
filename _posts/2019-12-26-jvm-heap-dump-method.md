---
layout: post
title: 生成Heap Dump的几种方式
date: 2019-12-26
categories:
    - jvm
comments: true
permalink: jvm-heap-dump-method.html
---

Heap Dump 包含的信息

- 所有的对象信息
- 对象的类信息、字段信息、原生值(int, long等)及引用值
- 所有的类信息
- 类加载器、类名、超类及静态字段
- 垃圾回收的根对象
- 根对象是指那些可以直接被虚拟机触及的对象
- 线程栈及局部变量
- 包含了转储时刻的线程调用栈信息和栈帧中的局部变量信息

# 使用 jmap 命令生成 dump 文件

```
jmap -dump:live,format=b,file=d:\dump\heap.hprof <pid>
```


# 使用 jcmd 命令生成 dump 文件

```
jcmd <pid> GC.heap_dump d:\dump\heap.hprof
```

# 使用 JVM 参数获取 dump 文件

- `-XX:+HeapDumpOnOutOfMemoryError` 当OutOfMemoryError发生时自动生成 Heap Dump 文件
- `-XX:+HeapDumpBeforeFullGC` 当 JVM 执行 FullGC 前执行 dump
- `-XX:+HeapDumpAfterFullGC`   当 JVM 执行 FullGC 后执行 dump
- `-XX:+HeapDumpOnCtrlBreak`  交互式获取dump。在控制台按下快捷键Ctrl + Break时，JVM就会转存一下堆快照。
- `-XX:HeapDumpPath=d:\test.hprof` 指定 dump 文件存储路径。


