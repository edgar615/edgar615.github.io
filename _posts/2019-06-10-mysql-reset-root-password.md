---
layout: post
title: MySQL重置root密码
date: 2019-06-10
categories:
    - MySQL
comments: true
permalink: mysql-reset-root-password.html
---

```
mysqladmin -u root -p password ex

GRANT ALL PRIVILEGES ON *.* TO 'root'@'%'IDENTIFIED BY '123456' WITH GRANT

flush privileges;
```


远程访问

```
mysql> use mysql;
Database changed
mysql> grant all privileges  on *.* to root@'%' identified by '新密码';
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
```

忘记root密码

1.修改my.cnf，在mysqld增加一行

```
skip-grant-tables
```

2.重启MySQL后root登录（不需要密码）

```
use mysql;
UPDATE user SET authentication_string = password('新密码') WHERE User = 'root';#5.7+
UPDATE user SET password = password('新密码') WHERE User = 'root';#5.6-
flush privileges;
```

3.删除第一步增加的`skip-grant-tables`重启MySQL


错误

```
You must reset your password using ALTER USER statement before executing this statement.
```

```
SET PASSWORD = PASSWORD('your new password');

ALTER USER 'root'@'localhost' PASSWORD EXPIRE NEVER;

flush privileges;
```


**最新版的MySQL已经删除了password函数**
在`skip-grant-tables`模式下执行

```sql
update user set authentication_string = ''  where user='root' ;   
```

关闭`skip-grant-tables`模式

重新登录root（密码为空）

执行sql

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY '新密码';
```
