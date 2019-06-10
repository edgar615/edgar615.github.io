---
layout: post
title: MySQL常用SQL
date: 2019-06-10
categories:
    - MySQL
comments: true
permalink: mysql-common-sql.html
---

# 修改列

- 查看列：desc 表名;
- 修改表名：alter table 原表名rename to 新表名; 
- 添加列：alter table 表名 add column 列名 类型; 
- 删除列：alter table 表名 drop column 列名; 
- 修改列名MySQL： alter table 表名 change 原列名 新列名 类型; 
- 修改列属性：alter table t_book modify name varchar(22); 
- 新增字段到第一列：alter table 表名 add column 列名 类型 FIRST;
- 新增字段到某列后面：alter table 表名 add column 列名 类型 AFTER 列名;
- 修改默认值：ALTER TABLE 表名 ALTER 列名 SET DEFAULT 默认值;
- 删除默认值：ALTER TABLE 表名 ALTER 列名 DROP DEFAULT;
- not null: ALTER TABLE 表名 MODIFY 列名 类型 NOT NULL DEFAULT 默认值;

FIRST 和 AFTER 关键字只占用于 ADD 子句，所以如果你想重置数据表字段的位置就需要先使用 DROP 删除字段然后使用 ADD 来添加字段并设置位置。

# 索引
## 创建索引
### ALTER TABLE

ALTER TABLE用来创建普通索引、UNIQUE索引或PRIMARY KEY索引。
```
ALTER TABLE table_name ADD INDEX index_name (column_list)
ALTER TABLE table_name ADD UNIQUE (column_list)
ALTER TABLE table_name ADD PRIMARY KEY (column_list)
```
### CREATE INDEX
CREATE INDEX可对表增加普通索引或UNIQUE索引。
```
CREATE INDEX index_name ON table_name (column_list)
CREATE UNIQUE INDEX index_name ON table_name (column_list)
```
## 删除索引
可利用ALTER TABLE或DROP INDEX语句来删除索引。类似于CREATE INDEX语句，DROP INDEX可以在ALTER TABLE内部作为一条语句处理，语法如下。
```
DROP INDEX index_name ON talbe_name
ALTER TABLE table_name DROP INDEX index_name
ALTER TABLE table_name DROP PRIMARY KEY
```
## 查看索引
```
show index from tblname;
show keys from tblname;
```

# 新建数据库和用户
1.创建数据库
```
CREATE DATABASE IF NOT EXISTS DB_NAME ;
```
2.创建用户
```
CREATE USER 'USERNAME'@'%' IDENTIFIED BY 'PASSWORD' ;
```
3.授权
格式
```
GRANT privileges ON databasename.tablename TO 'username'@'host' 
```
授权对某个数据库的所有操作
```
GRANT ALL ON DB_NAME.* TO 'USERNAME'@'%' ;
```
```
GRANT SELECT, INSERT ON test.user TO 'pig'@'%';
GRANT ALL ON *.* TO 'pig'@'%';
```
注意:用以上命令授权的用户不能给其它用户授权,如果想让该用户可以授权,用以下命令:
```
GRANT privileges ON databasename.tablename TO 'username'@'host' WITH GRANT OPTION
```
4.刷新系统权限
```
FLUSH PRIVILEGES ;
```

5.设置与更改用户密码

```
SET PASSWORD FOR 'username'@'host' = PASSWORD('newpassword');
```

如果是当前登陆用户用
```SET PASSWORD = PASSWORD("newpassword");
```

例子: 
```
SET PASSWORD FOR 'pig'@'%' = PASSWORD("123456");
```

6.撤销用户权限

命令: 
```
REVOKE privilege ON databasename.tablename FROM 'username'@'host';
```
说明: privilege, databasename, tablename - 同授权部分.

例子: REVOKE SELECT ON *.* FROM 'pig'@'%';

注意: 假如你在给用户'pig'@'%'授权的时候是这样的(或类似的):GRANT SELECT ON test.user TO 'pig'@'%', 则在使用REVOKE SELECT ON *.* FROM 'pig'@'%';命令并不能撤销该用户对test数据库中user表的SELECT 操作.相反,如果授权使用的是GRANT SELECT ON *.* TO 'pig'@'%';则REVOKE SELECT ON test.user FROM 'pig'@'%';命令也不能撤销该用户对test数据库中user表的Select 权限.

具体信息可以用命令SHOW GRANTS FOR 'pig'@'%'; 查看. 

7.删除用户

命令: 
```
DROP USER 'username'@'host'; 
```


操作权限：

| `ALTER`                  | Allows use of `ALTER TABLE`.             |
| ------------------------ | ---------------------------------------- |
| `ALTER ROUTINE`          | Alters or drops stored routines.         |
| `CREATE`                 | Allows use of `CREATE TABLE`.            |
| `CREATE ROUTINE`         | Creates stored routines.                 |
| `CREATE TEMPORARY TABLE` | Allows use of `CREATE TEMPORARY TABLE`.  |
| `CREATE USER`            | Allows use of `CREATE USER`, `DROP USER`, `RENAME USER`, and `REVOKE ALL PRIVILEGES`. |
| `CREATE VIEW`            | Allows use of `CREATE VIEW`.             |
| `DELETE`                 | Allows use of `DELETE`.                  |
| `DROP`                   | Allows use of `DROP TABLE`.              |
| `EXECUTE`                | Allows the user to run stored routines.  |
| `FILE`                   | Allows use of `SELECT`... `INTO OUTFILE` and `LOAD DATA INFILE`. |
| `INDEX`                  | Allows use of `CREATE INDEX` and `DROP INDEX`. |
| `INSERT`                 | Allows use of `INSERT`.                  |
| `LOCK TABLES`            | Allows use of `LOCK TABLES` on tables for which the user also has `SELECT` privileges. |
| `PROCESS`                | Allows use of `SHOW FULL PROCESSLIST`.   |
| `RELOAD`                 | Allows use of `FLUSH`.                   |
| `REPLICATION`            | Allows the user to ask where slave or master |
| `CLIENT`                 | servers are.                             |
| `REPLICATION SLAVE`      | Needed for replication slaves.           |
| `SELECT`                 | Allows use of `SELECT`.                  |
| `SHOW DATABASES`         | Allows use of `SHOW DATABASES`.          |
| `SHOW VIEW`              | Allows use of `SHOW CREATE VIEW`.        |
| `SHUTDOWN`               | Allows use of `mysqladmin shutdown`.     |
| `SUPER`                  | Allows use of `CHANGE MASTER`, `KILL`, `PURGE MASTER LOGS`, and `SET GLOBAL` SQL statements. Allows `mysqladmin debug` command. Allows one extra connection to be made if maximum connections are reached. |
| `UPDATE`                 | Allows use of `UPDATE`.                  |
| `USAGE`                  | Allows connection without any specific privileges. |