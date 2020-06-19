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
2. 使用batch，一次插入10000条
3. setAutoCommit(false)，一条条插入
4. setAutoCommit(false)，使用batch，一次插入10000条

分别插入1K，1W，10W，50W，100W，500W，1000W数据

BUFFER配置512M

结果如下

**一条条插入**

- 1K: 1625ms
- 1W: 4231ms
- 10W: 92044ms
- 50W: 464579ms
- 100W:

**使用batch，一次插入10000条**

- 1K: 2562ms
- 1W: 14433ms
- 10W: 119421ms
- 50W: 574060ms
- 100W:
- 500W:
- 1000W: 

**setAutoCommit(false)，一条条插入**

- 1K: 854ms
- 1W: 4738ms
- 10W: 29980ms
- 50W: 13852ms
- 100W:
- 500W:
- 1000W: 


**setAutoCommit(false)，使用batch，一次插入10000条**

- 1K: 1439ms
- 1W: 7853ms
- 10W: 52930ms
- 50W: 251585ms
- 100W:
- 500W:
- 1000W: 
