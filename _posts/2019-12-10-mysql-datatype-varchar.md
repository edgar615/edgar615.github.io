---
layout: post
title: MySQL数据类型-数据库varchar长度最佳实践（part4）
date: 2019-12-10
categories:
    - MySQL
comments: true
permalink: mysql-datatype-varchar.html
---

在很多文章里都看到说将VARCHAR设置为255，最近对原因搜索了一番，记录一下，对底层不太深入可能有错误

> `VARCHAR(3)`表示的是这一列最多存3个字符而不是3个字节，比如可以存“一二三”，实际存储时是编码为utf-8的。

对于InnoDB和MyISAM引擎的标准行格式（dynamic/compact）来说，`VARCHAR(50)`和`VARCHAR(255)`在存储方式上是没有区别的，都是1个字节表示字符串长度和字符串经编码后的字节（字节长度依赖于字符串的编码，每个字符可能占1到4个字节） 。

而`VARCHAR(256)`或更大，则需要两个字节来存储字符串长度。

> mysql5.0.3以前的版本varchar的最大长度就是255字节，之后是65535字节

但事实上，把所有较短的字符串列都设为`VARCHAR(255)`并不是最好的做法。尽管InnoDB是动态存储的，但别的数据库引擎不一定是如此。有的可能会使用固定长度的行，或者固定大小的内存表。内存表即为sql查询中产生的临时表。它通常会为varchar类型分配最大的空间，比如utf-8编码下，内存表可能要为`VARCHAR(255)`分配2+3×255字节（2是因为存的是字节长度而不是字符长度），如果行数非常多，这也会带来性能问题。不管其中每一行存储的数据是长还是短。

另外也InnoDB单列索引长度不能超过767bytes，联合索引还有一个限制是3072bytes，因此如果使用`VARCHAR(255)`且utf8编码，我们可能无法完全索引这个字段

> INNODB会使用编码上限计算，即每个字符占用3字节空间。两个字节存储长度
>
> 所以在utf8编码中，能使用完整索引的最大varchar长度为255（255*3=765）
>
> 在utf8mb4编码中，INNODB会使用4字节计算，所以能使用完整索引的最大varchar长度为191（191*4=764）

InnoDB最大的行的大小是半个database  page（大约8000字节），如果可变长的列（如varbinary、varchar、text、blob）超过了这个大小会被存到外面去，行里面只是存一个指针。这会比存inline慢很多。

所以结论是，我们应该用尽可能小的类型而不是统一用`VARCHAR(255)`。

参考资料

https://dba.stackexchange.com/questions/76469/mysql-varchar-length-and-performance

https://segmentfault.com/a/1190000002736763