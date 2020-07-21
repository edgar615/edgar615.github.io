---
layout: post
title: java插入大量数据到MySQL
date: 2020-06-19
categories:
    - java
comments: true
permalink: java-insert-large-data-mysql.html
---

> 测试失败。后面再找原因，o(╥﹏╥)o

测试了四张插入方式

1. 一条一条插入
2. 使用batch
3. setAutoCommit(false)，一条条插入
4. setAutoCommit(false)，使用batch

分别插入1K，1W，10W，50W，100W，500W，1000W数据

在Buffer比较小的时候，数据偏差较大，将Buffer设为5G，ChangeBuffer设为50%

结果如下

**一条条插入**

- 1K: 2180ms
- 1W: 15650ms
- 10W: 155154ms
- 50W: 752434ms
- 100W:

**使用batch**

- 1K: 1930ms
- 1W: 18404ms
- 10W: 168810ms
- 50W: 835741ms
- 100W:
- 500W:
- 1000W: 

**setAutoCommit(false)，一条条插入**

- 1K: 262ms
- 1W: 4738ms
- 10W: 24146ms
- 50W: 118544ms
- 100W:
- 500W:
- 1000W: 


**setAutoCommit(false)，使用batch**

- 1K: 505ms
- 1W: 4394ms
- 10W: 42241ms
- 50W: 208635ms
- 100W:
- 500W:
- 1000W: 
