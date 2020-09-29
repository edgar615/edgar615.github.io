---
layout: post
title: MySQL数据类型（3）-JSON
date: 2019-06-13
categories:
    - MySQL
comments: true
permalink: mysql-datatype-json.html
---

JSON数组和对象可以包含标量值是字符串或数字，JSON null文本，或者JSON布尔真或假的文字。在JSON对象键必须是字符串。时间（日期，时间或日期）标值也是允许的

# 1. 创建JSON列

```
CREATE TABLE test ( id INT NOT NULL auto_increment, ext json, PRIMARY KEY ( id ) );
```
# 2. 插入
```
INSERT INTO test ( ext )
VALUES
	( '{"name":"Edgar","age":30}' ),
	( '{"name":"Leona","age":30}' );
```
# 3. MySQL的JSON构造函数
**JSON_ARRAY**

```
INSERT INTO test ( ext )
VALUES
	( JSON_ARRAY ( 1, "abc", NULL, TRUE, CURTIME( ) ) );
```
[1, "abc", null, true, "03:02:12.000000"]

查询
```
SELECT JSON_EXTRACT(ext, '$[1]') FROM test where id = 3
--返回 "abc"
```

**JSON_OBJECT**
```
INSERT INTO test ( ext )
VALUES
	( JSON_OBJECT ( 'id', 87, 'name', 'carrot' ) );
```
{"id": 87, "name": "carrot"}

**JSON_MERGE**

```
INSERT INTO test ( ext )
VALUES
	( JSON_MERGE('[1, 2]', '[true, false]')),
	(JSON_MERGE('{"name": "x"}', '{"id": 47}')),
	(JSON_MERGE('1', 'true')),
	(JSON_MERGE('[1, 2]', '{"id": 47}'));
```
[1, 2, true, false]
{"id": 47, "name": "x"}
[1, true]
[1, 2, {"id": 47}]

查询
```
SELECT JSON_EXTRACT(ext, '$[2].id') FROM test where id = 8
```

# 4. 提取JSON值
```
SELECT
	id,
	json_extract ( ext, '$.name' ) AS NAME,
	json_extract ( ext, '$.age' ) age 
FROM
	test;
	
SELECT JSON_EXTRACT(ext, '$[1]') FROM test where id = 3
SELECT JSON_EXTRACT(ext, '$[2].id') FROM test where id = 8
```
ext->'$.age'与等效
```
SELECT
	id,
	json_extract ( ext, '$.name' ) AS NAME,
	ext->'$.age' age
FROM
	test;
```
# 5. 提取JSON的KEY
```
SELECT
	id,
	json_keys ( ext ) 
FROM
	test
```
返回的json_keys是一个数组
# 6. JSON_INSERT
增加了新的值，但不会取代现有的值
```
UPDATE test 
SET ext = JSON_INSERT( ext, '$.name', "Edgar615", '$.address', 'wuhan' ) 
WHERE
	id = 1;
```
执行完之后，JSON的name并没变，只是新加入了一个address
```
{"age": 30, "name": "Edgar", "address": "wuhan"}
```
# 7. JSON_SET 更新K-V
```
UPDATE test 
SET ext = JSON_SET( ext, '$.name', "Brena", '$.address', 'wuhan' ) 
WHERE
	id = 2;
```
执行完之后，JSON的name变了，并且新加入了一个address
```
{"age": 30, "name": "Brena", "address": "wuhan"}
```
# 8. JSON_REMOVE 删除KEY
```
UPDATE test 
SET ext = JSON_REMOVE( ext, '$.name', '$.address' ) 
WHERE
	id = 2;
```
执行完之后，JSON的只剩下{"age": 30}
```
UPDATE test 
SET ext = JSON_REMOVE( ext, '$[1]' ) 
WHERE
	id = 8;
	
UPDATE test 
SET ext = JSON_REMOVE( ext, '$[1].id' ) 
WHERE
	id = 8;
```
[1, {}]

# 9. JSON_REPLACE
替换现有的值，并忽略新值：
```
UPDATE test 
SET ext = JSON_REPLACE( ext, '$.age', 10, '$.name', 'Leona' ) 
WHERE
	id = 2;
```
# 10. JSON_TYPE
返回JSON的类型
```
SELECT json_type(ext) from test
```


# 11. 比较
JSON值可使用进行比较 =，<，<=，>，> =，<>，！=和 <=> 运算。 
http://www.huzs.net/?p=2318


# 12. 查询条件
```
SELECT * from test where json_extract(ext,'$.age') > 10
SELECT * from test where ext->'$.age' > 10
```
上面的查询会全表扫描，为了提高效率，需要对检索的key创建虚拟列，然后再虚拟列上创建索引
```
EXPLAIN SELECT * from test where ext->'$.age' > 10
```
创建虚拟列
```
ALTER TABLE test ADD age_virtual int GENERATED ALWAYS AS (ext->'$.age') VIRTUAL;
```
对虚拟列创建索引
```
CREATE INDEX age_virtual_index  ON test(age_virtual);
```
WHERE中使用虚拟列
```
SELECT * from test where age_virtual > 10
```
JSON_CONTAINS

```
select * from commodity where json_contains(ext, JSON_ARRAY (1), '$.physique')
```

