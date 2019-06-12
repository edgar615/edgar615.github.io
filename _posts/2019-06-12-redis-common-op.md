---
layout: post
title: redis常用命令
date: 2019-06-12
categories:
    - redis
comments: true
permalink: redis-common-op.html
---

# 统计个数

redis中名称含有OMP_OFFLINE的key的个数；

```
src/redis-cli keys "*OMP_OFFLINE*" | wc -l
```

# 批量删除
批量删除 0号数据库中名称含有OMP_OFFLINE的key：

```
src/redis-cli -n 0 keys "*OMP_OFFLINE*" | xargs src/redis-cli -n 0 del
```
