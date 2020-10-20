---
layout: post
title: linux 根目录爆满 解决 /dev/mapper/centos-root 100%问题
date: 2019-10-01
categories:
    - linux
comments: true
permalink: linux-root-space.html
---

使用 `df -h` 命令查看，发现/根目录的剩余空间为0

```
~# df -h
Filesystem      				  Size  Used Avail Use% Mounted on
/dev/mapper/centos-root            30G  30K  0G   100% /
...
```

使用 du -h -x --max-depth=1 查看哪个目录占用过高，对于过高目录中的内容适当删减腾出一些空间

```
~# du -h -x --max-depth=1 /
...
15G    /var
```

发现是`/var`目录占用过高，切换到`/var`目录里继续查找，发现是`/var/log/messaage`占用过高

