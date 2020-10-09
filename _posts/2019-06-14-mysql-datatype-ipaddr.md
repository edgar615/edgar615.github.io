---
layout: post
title: MySQL数据类型（5）-IP地址的存储
date: 2019-06-14
categories:
    - MySQL
comments: true
permalink: mysql-datatype-ipaddr.html
---

用 int32 来存放 IPv4 地址，比单纯用字符串节省空间。

简单的对比占用磁盘空间大小，我定义了四张表来存储IPV4。

```
create table ip_int_unsigned( 
	ipaddr int(11) unsigned
);

create table ip_int( 
	ipaddr int(11)
);

create table ip_long( 
	ipaddr bigint(20)
);


create table ip_char( 
	ipaddr varchar(15)
);
```

四个表个30万数据，查看磁盘空间占用

```
~# ls -sish /usr/local/mysql/data/test/ip*
2259863 20M /usr/local/mysql/data/test/ip_char.ibd  
2259859 18M /usr/local/mysql/data/test/ip_int_unsigned.ibd
2259861 18M /usr/local/mysql/data/test/ip_int.ibd   
2259862 19M /usr/local/mysql/data/test/ip_long.ibd
```

可以看到char占用的空间更多

可以使用`inet_aton`和`inet_ntoa`来转换IP地址和整型

```
mysql> select inet_aton('192.16.10.10');
+---------------------------+
| inet_aton('192.16.10.10') |
+---------------------------+
|                3222276618 |
+---------------------------+
1 row in set (0.00 sec)

mysql> select inet_ntoa(3222276618);
+-----------------------+
| inet_ntoa(3222276618) |
+-----------------------+
| 192.16.10.10          |
+-----------------------+
1 row in set (0.00 sec)
```



