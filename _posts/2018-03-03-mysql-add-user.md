---
layout: post
title: MySQL添加代码
date: 2018-03-03
categories:
    - MySQL
comments: true
permalink: mysql-add-user.html
---

```
mysql> create user 'root'@'%' IDENTIFIED BY '123456';
Query OK, 0 rows affected (0.01 sec)

mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%';
Query OK, 0 rows affected (0.01 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.01 sec)
```
