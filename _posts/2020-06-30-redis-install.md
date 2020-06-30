---
layout: post
title: redis安装问题
date: 2020-06-30
categories:
    - redis
comments: true
permalink: redis-install.html
---

记录一些安装redis遇到的问题

**1. 出现cc: command not found**

安装没有安装gcc环境，需要安装gcc环境

```
yum install gcc
```

**2. 报错 jemalloc/jemalloc.h: No such file or directory**

网上大部分解决办法：`make MALLOC=libc`,虽然能解决问题，但是指定内存分配器为 libc。

然而jemalloc 内存分配器在实践中处理内存碎片是要比libc 好的，所以我们还是要用jemalloc内存分配器。

这个错误的本质是我们在开始执行make 时遇到了错误（大部分是由于gcc未安装），然后我们安装好了gcc 后，我们再执行make ,这时就出现了`jemalloc/jemalloc.h: No such file or directory`。

这是因为上次的编译失败，有残留的文件，我们需要清理下，然后重新编译就可以了。

```
make distclean  && make
```
