---
layout: post
title: MySQL事务（3）- 当前读
date: 2019-06-15
categories:
    - MySQL
comments: true
permalink: mysql-transaction-3.html
---

这个题目来着极客时间的收费课程，首先要说明一个小知识，事务一致性视图ReadView的创建时间有两种

- 在执行第一个快照读语句时创建的.
- 在执行 `start transaction with consistent snapshot;`后创建

先准备数据

```
CREATE TABLE `test`  (
  `a` smallint(5) UNSIGNED NOT NULL AUTO_INCREMENT,
  `b` smallint(5) NULL DEFAULT NULL,
  PRIMARY KEY (`a`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci;

INSERT INTO `test` VALUES (1, 1);
INSERT INTO `test` VALUES (2, 2);
INSERT INTO `test` VALUES (3, 3);
INSERT INTO `test` VALUES (4, 4);
```

开启3个会话，按顺序执行下面语句

1. 事务A
```
start transaction with consistent snapshot;
```

2. 事务B
```
start transaction with consistent snapshot;
```

3. 事务C
```
update test set b = b + 1 where a = 1;
```

4. 事务B
```
update test set b = b + 1 where a = 1;
select * from test where a = 1;
+---+---+
| a | b |
+---+---+
| 1 | 3 |
+---+---+
```

5. 事务A
```
select * from test where a = 1;
+---+---+
| a | b |
+---+---+
| 1 | 1 |
+---+---+
```
为什么第四步事务B返回的是3而不是2?**因为`update test set b = b + 1 where a = 1;`不是快照读而是当前读**

# 当前读
**当前读**  读取的是最新版本, 并且对读取的记录加锁, 阻塞其他事务同时改动相同记录，避免出现安全问题。

下面的语句使用的是当前读

- select...lock in share mode (共享读锁)
- select...for update
- update
- delete
- insert

假设要update一条记录，但是另一个事务已经delete这条数据并且commit了，如果不加锁就会产生冲突。所以update的时候肯定要是当前读，得到最新的信息并且锁定相应的记录。

对于update的过程，首先会执行当前读，然后把返回的数据加锁，之后执行update。加锁是防止别的事务在这个时候对这条记录做什么，默认加的是排他锁，这样就可以保证数据不会出错了。但注意一点，就算这里加了写锁，别的事务也还是能访问的，因为数据库采取了**一致性非锁定读**，别的事务会去读取一个快照数据。

如果我们将上面的事务C做一点改动，在update后不立即提交，这时事务B的结果有什么不同呢？
3. 事务C
```
start transaction with consistent snapshot;
update test set b = b + 1 where a = 1;
```
因为事务B是当前读，必须要读最新版本，而且必须加锁，而事务C的写锁还没有释放，所以事务B会等待事务C释放说完

如果我们对事务A在做一点改动
```
select * from test where a = 1;
+---+---+
| a | b |
+---+---+
| 1 | 3 |
+---+---+

select * from test where a = 1 lock in share mode;
+---+---+
| a | b |
+---+---+
| 1 | 5 |
+---+---+
1 row in set (16.01 sec)

select * from test where a = 1;
+---+---+
| a | b |
+---+---+
| 1 | 3 |
+---+---+
1 row in set (0.06 sec)
```
可以看到`select * from test where a = 1;`依然使用的是快照读
