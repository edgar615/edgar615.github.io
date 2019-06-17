---
layout: post
title: MySQL分页优化
date: 2019-06-18
categories:
    - MySQL
comments: true
permalink: mysql-pagination-optimize.html
---

对于MySQL分页，我们最常用的方式是

1. 计算总数 `select count(*) from table where ...`
2. 计算得出偏移量`select * from table where ... limit 100,10`

上面的语句在数据量小的时候没有问题，但在数量大的时候会存在明显的性能问题

