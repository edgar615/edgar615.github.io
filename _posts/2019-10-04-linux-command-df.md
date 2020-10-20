---
layout: post
title: linux命令-df
date: 2019-10-04
categories:
    - linux
comments: true
permalink: linux-command-df.html
---

**df命令**用于显示磁盘分区上的可使用的磁盘空间。默认显示单位为KB。可以利用该命令来获取硬盘被占用了多少空间，目前还剩下多少空间等信息。

```
-a或--all：包含全部的文件系统；
--block-size=<区块大小>：以指定的区块大小来显示区块数目；
-h或--human-readable：以可读性较高的方式来显示信息；
-H或--si：与-h参数相同，但在计算时是以1000 Bytes为换算单位而非1024 Bytes；
-i或--inodes：显示inode的信息；
-k或--kilobytes：指定区块大小为1024字节；
-l或--local：仅显示本地端的文件系统；
-m或--megabytes：指定区块大小为1048576字节；
--no-sync：在取得磁盘使用信息前，不要执行sync指令，此为预设值；
-P或--portability：使用POSIX的输出格式；
--sync：在取得磁盘使用信息前，先执行sync指令；
-t<文件系统类型>或--type=<文件系统类型>：仅显示指定文件系统类型的磁盘信息；
-T或--print-type：显示文件系统的类型；
-x<文件系统类型>或--exclude-type=<文件系统类型>：不要显示指定文件系统类型的磁盘信息；
--help：显示帮助；
--version：显示版本信息。
```

- **查看系统磁盘设备，默认是KB为单位**

```
~# df
Filesystem     1K-blocks     Used Available Use% Mounted on
udev             2012860        4   2012856   1% /dev
tmpfs             404816    28288    376528   7% /run
/dev/vda1       41151808 20880136  18158244  54% /
none                   4        0         4   0% /sys/fs/cgroup
none                5120        0      5120   0% /run/lock
none             2024072        0   2024072   0% /run/shm
none              102400        4    102396   1% /run/user
```

- **使用`-h`选项以KB以上的单位来显示，可读性高**

```
~# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            2.0G  4.0K  2.0G   1% /dev
tmpfs           396M   28M  368M   7% /run
/dev/vda1        40G   20G   18G  54% /
none            4.0K     0  4.0K   0% /sys/fs/cgroup
none            5.0M     0  5.0M   0% /run/lock
none            2.0G     0  2.0G   0% /run/shm
none            100M  4.0K  100M   1% /run/user
```

- **查看全部文件系统**

```
root@iZuf6fbha9mr8eysoj3mabZ:~# df -ha
Filesystem      Size  Used Avail Use% Mounted on
sysfs              0     0     0    - /sys
proc               0     0     0    - /proc
udev            2.0G  4.0K  2.0G   1% /dev
devpts             0     0     0    - /dev/pts
tmpfs           396M   28M  368M   7% /run
/dev/vda1        40G   20G   18G  54% /
none            4.0K     0  4.0K   0% /sys/fs/cgroup
none               0     0     0    - /sys/fs/fuse/connections
....
```

