---
layout: post
title: MySQL安装与使用（2）- 重置root密码、添加用户
date: 2018-03-02
categories:
    - MySQL
comments: true
permalink: mysql-reset-root-password.html
---

# 1. 重置密码
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

```
update user set authentication_string = ''  where user='root' ;   
```

关闭`skip-grant-tables`模式

重新登录root（密码为空）

执行sql

```
ALTER USER 'root'@'localhost' IDENTIFIED BY '新密码';
```

# 2. 添加用户

添加用户

```
//只允许指定ip连接
create user '新用户名'@'localhost' identified by '密码';
//允许所有ip连接（用通配符%表示）
create user '新用户名'@'%' identified by '密码';
```

为新用户授权

```
//基本格式如下
grant all privileges on 数据库名.表名 to '新用户名'@'指定ip' identified by '新用户密码' ;
//示例
//允许访问所有数据库下的所有表
grant all privileges on *.* to '新用户名'@'指定ip' identified by '新用户密码' ;
//指定数据库下的指定表
grant all privileges on test.test to '新用户名'@'指定ip' identified by '新用户密码' ;
```

设置用户权限

```
//设置用户拥有所有权限也就是管理员
grant all privileges on *.* to '新用户名'@'指定ip' identified by '新用户密码' WITH GRANT OPTION;
//拥有查询权限
grant select on *.* to '新用户名'@'指定ip' identified by '新用户密码' WITH GRANT OPTION;
//其它操作权限说明,select查询 insert插入 delete删除 update修改
//设置用户拥有查询插入的权限
grant select,insert on *.* to '新用户名'@'指定ip' identified by '新用户密码' WITH GRANT OPTION;
//取消用户查询的查询权限
REVOKE select ON what FROM '新用户名';
```

删除用户

```
DROP USER username@localhost;
```

修改后刷新权限

```
FLUSH PRIVILEGES;
```

# 3. 权限

操作权限：

| `ALTER`                  | Allows use of `ALTER TABLE`.                                 |
| ------------------------ | ------------------------------------------------------------ |
| `ALTER ROUTINE`          | Alters or drops stored routines.                             |
| `CREATE`                 | Allows use of `CREATE TABLE`.                                |
| `CREATE ROUTINE`         | Creates stored routines.                                     |
| `CREATE TEMPORARY TABLE` | Allows use of `CREATE TEMPORARY TABLE`.                      |
| `CREATE USER`            | Allows use of `CREATE USER`, `DROP USER`, `RENAME USER`, and `REVOKE ALL PRIVILEGES`. |
| `CREATE VIEW`            | Allows use of `CREATE VIEW`.                                 |
| `DELETE`                 | Allows use of `DELETE`.                                      |
| `DROP`                   | Allows use of `DROP TABLE`.                                  |
| `EXECUTE`                | Allows the user to run stored routines.                      |
| `FILE`                   | Allows use of `SELECT`... `INTO OUTFILE` and `LOAD DATA INFILE`. |
| `INDEX`                  | Allows use of `CREATE INDEX` and `DROP INDEX`.               |
| `INSERT`                 | Allows use of `INSERT`.                                      |
| `LOCK TABLES`            | Allows use of `LOCK TABLES` on tables for which the user also has `SELECT` privileges. |
| `PROCESS`                | Allows use of `SHOW FULL PROCESSLIST`.                       |
| `RELOAD`                 | Allows use of `FLUSH`.                                       |
| `REPLICATION`            | Allows the user to ask where slave or master                 |
| `CLIENT`                 | servers are.                                                 |
| `REPLICATION SLAVE`      | Needed for replication slaves.                               |
| `SELECT`                 | Allows use of `SELECT`.                                      |
| `SHOW DATABASES`         | Allows use of `SHOW DATABASES`.                              |
| `SHOW VIEW`              | Allows use of `SHOW CREATE VIEW`.                            |
| `SHUTDOWN`               | Allows use of `mysqladmin shutdown`.                         |
| `SUPER`                  | Allows use of `CHANGE MASTER`, `KILL`, `PURGE MASTER LOGS`, and `SET GLOBAL` SQL statements. Allows `mysqladmin debug` command. Allows one extra connection to be made if maximum connections are reached. |
| `UPDATE`                 | Allows use of `UPDATE`.                                      |
| `USAGE`                  | Allows connection without any specific privileges.           |