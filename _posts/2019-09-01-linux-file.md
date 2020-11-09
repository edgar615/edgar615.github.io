---
layout: post
title: Linux文件类型
date: 2019-09-01
categories:
    - linux
comments: true
permalink: linux-file.html
---

Linux 常见的文件类型有以下 7 种:

- 普通文件（比如一个文本文件）
- 目录文件（目录也是一个特殊的文件，它用来存储文件清单，比如/也是一个文件）
- 可执行文件（上面的rm就是一个可执行文件）
- 管道文件
- Socket 文件
- 软链接文件（相当于指向另一个文件所在路径的符号）
- 硬链接文件（相当于指向另一个文件的指针）

你如果使用ls -F就可以看到当前目录下的文件和它的类型

- * 结尾的是可执行文件；
- = 结尾的是 Socket 文件；
- @ 结尾的是软链接；
- | 结尾的管道文件；
- 没有符号结尾的是普通文件；
- / 结尾的是目录。

```
~# ls -F /
bin/   dev/  home/        initrd.img.old@  lib64/       media/  my/   proc/  run/   srv/  tmp/  var/      vmlinuz.old@
boot/  etc/  initrd.img@  lib/             lost+found/  mnt/    opt/  root/  sbin/  sys/  usr/  vmlinuz@  workspace/

~# ls -F /usr/bin/r*
/usr/bin/ranlib*           /usr/bin/renice*      /usr/bin/rgrep*    /usr/bin/rsh@              /usr/bin/rview@
/usr/bin/rcp@              /usr/bin/report-hw*   /usr/bin/rlogin@   /usr/bin/rsync*            /usr/bin/rvim@
/usr/bin/readelf*          /usr/bin/reset@       /usr/bin/rngtest*  /usr/bin/rtstat@
/usr/bin/recode-sr-latin*  /usr/bin/resizecons*  /usr/bin/routef*   /usr/bin/runcon*
/usr/bin/rename@           /usr/bin/resizepart*  /usr/bin/routel*   /usr/bin/run-mailcap*
/usr/bin/rename.ul*        /usr/bin/rev*         /usr/bin/rpcgen*   /usr/bin/run-with-aspell*

~# ls -F /usr/local/mysql
bin/   docs/     lib/     man/        mysqld.pid   mysql.sock.lock  share/
data/  include/  LICENSE  mysqld.err  mysql.sock=  README           support-files/
```

Socket 是网络插座，是客户端和服务器之间同步数据的接口。其实，Linux 不只把 Socket 抽象成了文件，设备基本也都被抽象成了文件。因为设备需要不断和操作系统交换数据。而交换方式只有两种——读和写。所以设备是可以抽象成文件的，因为文件也支持这两种操作。

Linux 把所有的设备都抽象成了文件，比如说打印机、USB、显卡等。这让整体的系统设计变得高度统一。