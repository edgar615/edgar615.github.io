---
layout: post
title: MySQL查询过程
date: 2019-10-10
categories:
    - MySQL
comments: true
permalink: mysql-query-process.html
---


# 1. MySQL逻辑架构

![](/assets/images/posts/mysql-query-process/mysql-query-process-1.png)

整张图由三部分组成，从上到下分别是客户端（紫色）、MySQL Server层（绿色）、MySQL存储引擎层（黄色）。

- 客户端不言而喻，主要负责与MySQL Server层建立连接，发送查询请求以及接受响应的结果集。
- MySQL Server层，主要包括连接器、查询缓存、分析器、优化器、执行器等。这些组件包含了MySQL的大部分主要功能，例如平时使用最多的存储过程、触发器、视图都在这一层中。还有一个通用的日志模块 bin log。
- MySQL 存储引擎层，主要负责数据的存储和提取。其支持多个存储引擎，例如：InnoDB、MyISAM等。中间的服务层通过API与存储引擎通信，这些API接口屏蔽了不同存储引擎间的差异。常用的有InnoDB，它从MySQL  5.5.5版本开始成为了MySQL的默认存储引擎，重要的是InnoDB 引擎包含了自带的日志模块 redo  log。

上面介绍了MySQL的组件结构，那么这里将其处理SQL语句的流程简单梳理一遍，之后再对每个组件逐一进行介绍。如下图所示，顺着编号来看看MySQL的各各组件是如何处理SQL查询请求的。

![](/assets/images/posts/mysql-query-process/mysql-query-process-2.png)

1. 连接器：当客户端登陆MySQL的时候，对身份认证和权限判断。
2. 查询缓存: 执行查询语句的时候，会先查询缓存（MySQL 8.0 版本后移除）。
3. 分析器: 假设在没有命中查询缓存的情况下，SQL请求就会来到分析器。分析器负责明确SQL要完成的功能，以及检查SQL的语法是否正确。
4. 优化器：为SQL提供优化执行的方案。
5. 执行器: 将语句分发到对应的存储引擎执行，并返回数据。

# 2. MySQL查询过程
在进行MySQL的优化之前，必须要了解的就是MySQL的查询过程，**很多查询优化工作实际上就是遵循一些原则，让MySQL的优化器能够按照预想的合理方式运行而已**。

## 2.1. 连接器

客户端需要通过连接器访问MySQL  Server，连接器主要负责身份认证和权限鉴别的工作。也就是负责用户登录数据库的相关认证操作，例如：校验账户密码，权限等。在用户名密码合法的前提下，会在权限表中查询用户对应的权限，并且将该权限分配给用户。在连接完成以后可以通过图3看到连接状态，可以通过命令行“show  processlist”生成下图的查询结果。其中“Command”列返回的内容中，“Sleep”表示MySQL相同中对应一个空闲连接。而“Query”表示正在查询的连接。

![](/assets/images/posts/mysql-query-process/mysql-query-process-3.png)

上面提到了连接状态，这里将5种连接状态整理为如下表格，方便大家参考。

| Command        | 含义                     |
| -------------- | ------------------------ |
| sleep          | 线程正在等待客户端发数据 |
| query          | 连接线程正在执行查询     |
| locked         | 线程正在等待表锁的释放   |
| sorting result | 线程正在对结果进行排序   |
| sending data   | 向请求端返回数据         |

 MySQL将连接器中的连接分为长连接和短连接。

- 长连接是指连接成功后，客户端请求一直使用是同一个连接。
- 短连接是指每次执行完SQL请求的操作之后会断开连接，如果再有SQL请求会重新建立连接。由于短连接会反复创建连接消耗相同资源，因此多数情况下会选择长连接。但是为了保持长连接，会占用系统内存，而这些被占用的内存知道连接断开以后才会释放。这里提出了两个解决方案：
  - 定期断开长连接，每隔一段时间或者执行一个占用内存的大查询以后断开连接，从而释放内存，当查询的时候再重新创建连接。 
  - MySQL 5.7 或者更高的版本，通过执行 mysql_reset_connection 来重新初始化连接。此过程不会重新建立连接，但是会释放占用的内存，将连接恢复到刚刚创立连接的状态。

**客户端/服务端通信协议**

MySQL客户端/服务端通信协议是“半双工”的：在任一时刻，要么是服务器向客户端发送数据，要么是客户端向服务器发送数据，这两个动作不能同时发生。一旦一端开始发送消息，另一端要接收完整个消息才能响应它，所以我们无法也无须将一个消息切成小块独立发送，也没有办法进行流量控制。

客户端用一个单独的数据包将查询请求发送给服务器，所以当查询语句很长的时候，需要设置max_allowed_packet参数。但是需要注意的是，如果查询实在是太大，服务端会拒绝接收更多数据并抛出异常。

与之相反的是，服务器响应给用户的数据通常会很多，由多个数据包组成。但是当服务器响应客户端请求时，客户端必须完整的接收整个返回结果，而不能简单的只取前面几条结果，然后让服务器停止发送。因而在实际开发中，**尽量保持查询简单且只返回必需的数据，减小通信间数据包的大小和数量是一个非常好的习惯，这也是查询中尽量避免使用SELECT *以及加上LIMIT限制的原因之一**。

## 2.2. 查询缓存

在解析一个查询语句前，如果查询缓存是打开的，那么MySQL会检查这个查询语句是否命中查询缓存中的数据。如果当前查询恰好命中查询缓存，在检查一次用户权限后直接返回缓存中的结果。这种情况下，查询不会被解析，也不会生成执行计划，更不会执行。

MySQL将缓存存放在一个引用表（不要理解成table，可以认为是类似于HashMap的数据结构），通过一个哈希值索引，这个哈希值通过查询本身、当前要查询的数据库、客户端协议版本号等一些可能影响结果的信息计算得来。所以两个查询在任何字符上的不同（例如：空格、注释），都会导致缓存不会命中。

如果查询中包含任何用户自定义函数、存储函数、用户变量、临时表、mysql库中的系统表，其查询结果都不会被缓存。比如函数`NOW()`或者`CURRENT_DATE()`会因为不同的查询时间，返回不同的查询结果，再比如包含`CURRENT_USER`或者`CONNECION_ID()`的查询语句会因为不同的用户而返回不同的结果，将这样的查询结果缓存起来没有任何的意义。

MySQL的查询缓存系统会跟踪查询中涉及的每个表，如果这些表（数据或结构）发生变化，那么和这张表相关的所有缓存数据都将失效。正因为如此，**在任何的写操作时，MySQL必须将对应表的所有缓存都设置为失效。如果查询缓存非常大或者碎片很多，这个操作就可能带来很大的系统消耗，甚至导致系统僵死一会儿。而且查询缓存对系统的额外消耗也不仅仅在写操作，读操作也不例外：**

- 任何的查询语句在开始之前都必须经过检查，即使这条SQL语句永远不会命中缓存
- 如果查询结果可以被缓存，那么执行完成后，会将结果存入缓存，也会带来额外的系统消耗

基于此，我们要知道并不是什么情况下查询缓存都会提高系统性能，缓存和失效都会带来额外消耗，只有当缓存带来的资源节约大于其本身消耗的资源时，才会给系统带来性能提升。如果系统确实存在一些性能问题，可以尝试打开查询缓存，并在数据库设计上做一些优化，比如：

- 用多个小表代替一个大表，注意不要过度设计
- 批量插入代替循环单条插入
- 合理控制缓存空间大小，一般来说其大小设置为几十兆比较合适
- 可以通过SQL_CACHE和SQL_NO_CACHE来控制某个查询语句是否需要进行缓存

注意：**不要轻易打开查询缓存，特别是写密集型应用**。如果你实在是忍不住，可以将query_cache_type设置为DEMAND，这时只有加入SQL_CACHE的查询才会走缓存，其他查询则不会，这样可以非常自由地控制哪些查询需要被缓存。

如果打开了缓存可以通过“show status like 'Qcache%'”命令查看缓存的情况。

![](/assets/images/posts/mysql-query-process/mysql-query-process-4.png)

其中几个使用较多的状态值如下：

- Qcache_inserts 是否有新的数据添加，每有一条数据添加Value会加一。
- Qcache_hits 查询语句是否命中缓存，每有一条语句命中Value会加一。
- Qcache_free_memory 缓存空闲大小。

**MySQL 8.0 版本后删除了查询缓存的功能，官方认为该功能应用场景较少，所以将其删除。**

由于QC需要缓存最新数据结果，因此表数据发生任何变化（INSERT、UPDATE、DELETE或其他可能产生数据变化的操作），都会导致QC被刷新。

**QC严格要求2次SQL请求要完全一样，包括SQL语句，连接的数据库、协议版本、字符集等因素都会影响**，下面几个例子中的SQL会被认为是完全不一样而不会使用同一个QC内存块：

```
mysql> set names latin1; SELECT * FROM table_name;

mysql> set names latin1; select * from table_name;

mysql> set names utf8; select * from table_name;
```

此外，QC也**不适用于**下面几个场景：

1、子查询或者外层查询；

2、存储过程、存储函数、触发器、event中调用的SQL，或者引用到这些结果的；

3、包含一些特殊函数时，例如：BENCHMARK()、CURDATE()、CURRENT_TIMESTAMP()、NOW()、RAND()、UUID()等等；

4、读取mysql、INFORMATION_SCHEMA、performance_schema 库数据的；

5、类似SELECT…LOCK IN SHARE MODE、SELECT…FOR UPDATE、SELECT..INTO OUTFILE/DUMPFILE、SELECT..WHRE…IS NULL等语句；

6、SELECT执行计划用到临时表（TEMPORARY TABLE）；

7、未引用任何表的查询，例如 SELECT 1+1 这种；

8、产生了 warnings 的查询；

9、SELECT语句里加了 SQL_NO_CACHE 关键字；

更加奇葩的是，MySQL在从QC中取回结果前，会先判断执行SQL的用户是否有全部库、表的SELECT权限，如果没有，则也不会使用QC。

相比下面这个，其实上面所说的都不重要。最为重要的是，在MySQL里QC是由一个**全局锁**在控制，**每次更新QC的内存块都需要进行锁定**。

例如，一次查询结果是20KB，当前 **query_cache_min_res_unit** 值设置为 4KB（默认值就是4KB，可调整），那么么本次查询结果共需要分为**5次**写入QC，**每次都要锁定**，可见其成本有多高。

## 2.3. 分析器

如果查询缓存没有命中，那么SQL请求会进入分析器，分析器是用来分辨SQL语句的执行目的，其执行过程大致分为两步：

- 第一步，词法分析(Lexical scanner)，主要负责从SQL 语句中提取关键字，比如：查询的表，字段名，查询条件等等。
- 第二步，语法规则(Grammar rule module)，主要判断SQL语句是否合乎MySQL的语法。

其实说白了词法分析(Lexical scanner) 就是将整个SQL语句拆分成一个个单词，而语法规则(Grammar rule module)则根据MySQL定义的语法规则生成对应的数据结构，并存储在对象结构当中。

其结果供优化器生成执行计划，再调用存储引擎接口执行。来看下面这个例子，假设有这样一个SQL语句“select username from userinfo”。

先通过词法分析，从左到右逐个字符进行解析，获得四个单词。

| 关键字 | 非关键字 | 关键字 | 非关键字 |
| ------ | -------- | ------ | -------- |
| select | username | from   | userinfo |

然后再通过语法规则解析，判断输入的SQL 语句是否满足MySQL语法，并且生成图5的语法树。由SQL语句生成的四个单词中，识别出两个关键字，分别是select  和from。根据MySQL的语法Select 和 from之间对应的是fields  字段，下面应该挂接username；在from后面跟随的是Tables字段，其下挂接的是userinfo。

![](/assets/images/posts/mysql-query-process/mysql-query-process-5.png)

## 2.4. 优化器

经过前面的步骤生成的语法树被认为是合法的了，并且由优化器将其转化成查询计划。多数情况下，一条查询可以有很多种执行方式，最后都返回相应的结果。优化器的作用就是找到这其中最好的执行计划。

MySQL使用基于成本的优化器，它尝试预测一个查询使用某种执行计划时的成本，并选择其中成本最小的一个。在MySQL可以通过查询当前会话的last_query_cost的值来得到其计算当前查询的成本。

```
mysql> select * from commodity limit 100, 10;
...
mysql> show status like 'last_query_cost';
+-----------------+-------------+
| Variable_name   | Value       |
+-----------------+-------------+
| Last_query_cost | 5063.599000 |
+-----------------+-------------+
1 row in set (0.13 sec)
```
示例中的结果表示优化器认为大概需要做5063个数据页的随机查找才能完成上面的查询。这个结果是根据一些列的统计信息计算得来的，**这些统计信息包括：每张表或者索引的页面个数、索引的基数、索引和数据行的长度、索引的分布情况等等。**

有非常多的原因会导致MySQL选择错误的执行计划，比如统计信息不准确、不会考虑不受其控制的操作成本（用户自定义函数、存储过程）、MySQL认为的最优跟我们想的不一样（我们希望执行时间尽可能短，但MySQL值选择它认为成本小的，但成本小并不意味着执行时间短）等等。

分析器生成的语法树作为优化器的输入，而优化器（黄色的部分）包含了逻辑变换和代价优化两部分的内容。在优化完成以后会生成SQL执行计划作为整个优化过程的输出，交给执行器在存储引擎上执行。

![](/assets/images/posts/mysql-query-process/mysql-query-process-6.png)

### 2.4.1. 逻辑变换

逻辑变换也就是在关系代数基础上进行变换，其目的是为了化简，同时保证SQL变化前后的结果一致，也就是逻辑变化并不会带来结果集的变化。其主要包括以下几个方面：

- 否定消除：针对表达式“和取”或“析取”前面出现“否定”的情况，应将关系条件进行拆分，从而将外层的“NOT”消除。
- 等值常量传递：利用了等值关系的传递特性，为了能够尽早执行“下推”运算。“下推”的基本策略是，始终将过滤表达式尽可能移至靠近数据源的位置。
- 常量表达式计算：对于能立刻计算出结果的表达式，直接计算结果，同时将结果与其他条件尽量提前进行化简。

这样讲概念或许有些抽象，通过下图来看看逻辑变化如何在SQL中执行的。

![](/assets/images/posts/mysql-query-process/mysql-query-process-7.png)

如上图所示，从上往下共有4个步骤：

1. 针对存在的SQL语句，首先通过“否定消除”，去掉条件判断中的“NOT”。语句由原来的“or”转换成“and”，并且大于小于符号进行变号。蓝色部分为修改前的SQL，红色是修改以后的SQL。
2. 等值传递，这一步很好理解分别降”t2.a=9” 和”t2.b=5”分别替换掉SQL中对应的值。
3. 接下来就是常量表达式计算，将“5+7”计算得到“12”。
4. 最后是常量表达式计算后的化简，将”9<=10”化简为”true”带入到最终的SQL表达式中完成优化。

### 2.4.2. **代价优化**

代价优化是用来确定每个表，根据条件是否应用索引，应用哪个索引和确定多表连接的顺序等问题。为了完成代价优化，需要找到一个代价最小的方案。

因此，优化器是通过基于代价的计算方法来决定如何执行查询的（Cost-based Optimization）。

简化的过程如下：

1. 赋值操作代价：针对每个数据库操作（创建表、返回数据集）设置对应的代价，这个代价值一般设置为1、0.2之类的值，没有具体的含义就是对操作的代价定义。
2. 计算操作数量：将SQL语句中涉及到的操作进行逻辑，并且做计算。说白了就是看这次SQL请求需要做哪些具体的数据库操作。
3. 求和操作代价：既然知道SQL由哪些数据库操作组成，同时知道每个操作对应的代价，求和以后就是知道整体SQL执行的代价。
4. 选择代价计划：如果说没给SQL执行的操作都是一个计划，那么这些操作的不同组合就会对应不同的计划，这里需要选择整体执行代价最低的操作计划，作为这次执行SQL语句的代价计划，从而达到总代价最低。

这里将配置操作的代价分为MySQL 服务层和MySQL 引擎层，MySQL 服务层主要是定义CPU的代价，而MySQL 引擎层主要定义IO代价。MySQL 5.7  引入了两个系统表mysql.server_cost和mysql.engine_cost来分别配置这两个层的代价，如下：

MySQL 服务层代价保存在表server_cost中，其具体内容如下：

- row_evaluate_cost (default 0.2) 计算符合条件的行的代价，行数越多，此项代价越大
- memory_temptable_create_cost (default 2.0) 内存临时表的创建代价
- memory_temptable_row_cost (default 0.2) 内存临时表的行代价
- key_compare_cost (default 0.1) 键比较的代价，例如排序
- disk_temptable_create_cost (default 40.0) 内部myisam或innodb临时表的创建代价
- disk_temptable_row_cost (default 1.0) 内部myisam或innodb临时表的行代价

由上可以看出创建临时表的代价是很高的，尤其是内部的myisam或innodb临时表。

MySQL 引擎层代价保存在表engine_cost中，其具体内容如下：

- io_block_read_cost (default 1.0) 从磁盘读数据的代价，对innodb来说，表示从磁盘读一个page的代价
- memory_block_read_cost (default 1.0) 从内存读数据的代价，对innodb来说，表示从buffer pool读一个page的代价

目前io_block_read_cost和memory_block_read_cost默认值均为1，实际生产中建议酌情调大memory_block_read_cost，特别是对普通硬盘的场景。

 MySQL会根据SQL查询生成的查询计划中对应的操作从上面两张代价表中查找对应的代价值，并且进行累加形成最终执行SQL计划的代价。再将多种可能的执行计划进行比较，选取最小代价的计划执行。

## 2.5. 执行器

当分析器生成查询计划，并且经过优化器以后，就到了执行器。执行器会选择执行计划开始执行，但在执行之前会校验请求用户是否拥有查询的权限，如果没有权限，就会返回错误信息，否则将会去调用MySQL引擎层的接口，执行对应的SQL语句并且返回结果。

例如SQL：“SELECT * FROM userinfo WHERE username = 'Tom';“

假设 “username“ 字段没有设置索引，就会调用存储引擎从第一条开始查，如果碰到了用户名字是” Tom“， 就将结果集返回，没有查找到就查看下一行，重复上一步的操作，直到读完整个表或者找到对应的记录。

需要注意SQL语句的执行顺序并不是按照书写顺序来的，顺序的定义会在分析器中做好，一般是按照如下顺序：

![](/assets/images/posts/mysql-query-process/mysql-query-process-8.png)


补充一个图

![](/assets/images/posts/mysql-optimize/mysql-optimize-2.png)

# 3. 参考资料

https://mp.weixin.qq.com/s/kKHOoB6WmYXp0crb-jxHSg

https://mp.weixin.qq.com/s/jg0Os6ZhiP7KwFd6LLkZ1A