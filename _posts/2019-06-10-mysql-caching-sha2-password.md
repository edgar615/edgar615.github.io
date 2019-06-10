---
layout: post
title: MySQL caching-sha2-password
date: 2019-06-10
categories:
    - MySQL
comments: true
permalink: mysql-caching-sha2-password.html
---
在安装mysql8的时候如果选择了密码加密，之后用客户端连接比如navicate，会提示客户端连接`caching-sha2-password`,是由于客户端不支持这种插件，可以通过如下方式进行修改：

```
mysql> alter user 'user'@'%' identified with mysql_native_password by 'password';
Query OK, 0 rows affected (0.09 sec)

mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)

```

