---
layout: post
title: MySQL重置root密码、添加用户
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