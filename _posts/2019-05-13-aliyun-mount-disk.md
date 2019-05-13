---
layout: post
title: 阿里云挂载云盘
date: 2019-05-13
categories:
    - 阿里云
comments: true
permalink: aliyun-mount-disk.html
---

查看数据盘

```
[root@izwz9247jeep0yhdmrbw3lz ~]# fdisk -l

Disk /dev/vda: 53.7 GB, 53687091200 bytes, 104857600 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x0008d73a

   Device Boot      Start         End      Blocks   Id  System
/dev/vda1   *        2048   104855551    52426752   83  Linux

Disk /dev/vdb: 223.3 GB, 223338299392 bytes, 436207616 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/vdc: 223.3 GB, 223338299392 bytes, 436207616 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/vdd: 536.9 GB, 536870912000 bytes, 1048576000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/vde: 536.9 GB, 536870912000 bytes, 1048576000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

格式化

```
mke2fs -O 64bit,has_journal,extents,huge_file,flex_bg,uninit_bg,dir_nlink,extra_isize /dev/vde
```

创建挂载点

```
mkdir /alidata
```

挂载

```
mount /dev/vde /alidata
```

如果需要分区，查看官方文档

https://help.aliyun.com/document_detail/34377.html?spm=a2c4g.11186623.6.663.12Ysut