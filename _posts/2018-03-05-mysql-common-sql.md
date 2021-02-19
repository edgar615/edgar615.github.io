---
layout: post
title: MySQL安装与使用（5）- 常用SQL
date: 2018-03-05
categories:
    - MySQL
comments: true
permalink: mysql-common-sql.html
---

# 1. 修改列

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

# 2. 索引
## 2.1. 创建索引
**ALTER TABLE**

ALTER TABLE用来创建普通索引、UNIQUE索引或PRIMARY KEY索引。
```
ALTER TABLE table_name ADD INDEX index_name (column_list)
ALTER TABLE table_name ADD UNIQUE (column_list)
ALTER TABLE table_name ADD PRIMARY KEY (column_list)
```
**CREATE INDEX**

CREATE INDEX可对表增加普通索引或UNIQUE索引。
```
CREATE INDEX index_name ON table_name (column_list)
CREATE UNIQUE INDEX index_name ON table_name (column_list)
```
## 2.3. 删除索引
可利用ALTER TABLE或DROP INDEX语句来删除索引。类似于CREATE INDEX语句，DROP INDEX可以在ALTER TABLE内部作为一条语句处理，语法如下。
```
DROP INDEX index_name ON talbe_name
ALTER TABLE table_name DROP INDEX index_name
ALTER TABLE table_name DROP PRIMARY KEY
```
## 2.4. 查看索引
```
show index from tblname;
show keys from tblname;
```