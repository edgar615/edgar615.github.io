---
layout: post
title: Docker - 容器技术原理
date: 2018-10-01
categories:
    - Docker
comments: true
permalink: docker-principle.html
---

# 1. chroot

> chroot 是在 Unix 和 Linux 系统的一个操作，针对正在运作的软件行程和它的子进程，改变它外显的根目录。一个运行在这个环境下，经由 chroot 设置根目录的程序，它不能够对这个指定根目录之外的文件进行访问动作，不能读取，也不能更改它的内容。
>
> ——维基百科

通俗地说 ，chroot 就是可以改变某进程的根目录，使这个程序不能访问目录之外的其他目录。

创建一个 rootfs 目录，拉取一个docker镜像解压到该目录下

```
$ cd rootfs 
$ docker export $(docker create busybox) -o busybox.tar
$ tar -xf busybox.tar
```

在 rootfs 目录下，我们会得到一些目录和文件

```
$ ls
bin  busybox.tar  dev  etc  home  proc  root  sys  tmp  usr  var
```

使用chroot运行一个sh进程，并把rootfs目录作为这个进程的根目录

```
# chroot /server/rootfs /bin/sh
```

可以使用ls命令查看当前目录

```
/ # pwd
/
/ # ls
/bin/sh: ls: not found
/ # /bin/ls
bin          dev          home         root         tmp          var
busybox.tar  etc          proc         sys          usr
```

这里可以看到当前进程的根目录已经变成了主机上的rootfs 目录。这样就实现了当前进程与主机的隔离。

查看路由信息

```
/ # /bin/ip route
default via 172.17.0.1 dev eth0
169.254.0.0/16 dev eth0 scope link  metric 1002
172.17.0.0/20 dev eth0 scope link  src 172.17.0.17
172.18.0.0/16 dev docker0 scope link  src 172.18.0.1
```

执行 ip route 命令后，你可以看到网络信息并没有隔离，实际上进程等信息此时也并未隔离。

要想实现一个完整的容器，我们还需要 Linux 的其他三项技术： Namespace、Cgroups 和联合文件系统。

Docker 是利用 Linux 的 Namespace 、Cgroups 和联合文件系统三大机制来保证实现的， 所以它的原理是使用  Namespace 做主机名、网络、PID 等资源的隔离，使用 Cgroups  对进程或者进程组做资源（例如：CPU、内存等）的限制，联合文件系统用于镜像构建和容器运行环境。

# 2. Namespace

Namespace 是 Linux  内核的一项功能，该功能对内核资源进行隔离，使得容器中的进程都可以在单独的命名空间中运行，并且只可以访问当前容器命名空间的资源。Namespace 可以隔离进程 ID、主机名、用户 ID、文件名、网络访问和进程间通信等相关资源。

Docker 主要用到以下五种命名空间。

- pid namespace：用于隔离进程 ID。
- net namespace：隔离网络接口，在虚拟的 net namespace 内用户可以拥有自己独立的 IP、路由、端口等。
- mnt namespace：文件系统挂载点隔离。
- ipc namespace：信号量,消息队列和共享内存的隔离。
- uts namespace：主机名和域名的隔离。

# 3. Cgroups

Cgroups 是一种 Linux 内核功能，可以限制和隔离进程的资源使用情况（CPU、内存、磁盘 I/O、网络等）。在容器的实现中，Cgroups 通常用来限制容器的 CPU 和内存等资源的使用。

# 4. 联合文件系统

联合文件系统，又叫  UnionFS，是一种通过创建文件层进程操作的文件系统，因此，联合文件系统非常轻快。Docker  使用联合文件系统为容器提供构建层，使得容器可以实现写时复制以及镜像的分层构建和存储。常用的联合文件系统有 AUFS、Overlay 和  Devicemapper 等。

# 5. 参考资料

《由浅入深吃透 Docker》