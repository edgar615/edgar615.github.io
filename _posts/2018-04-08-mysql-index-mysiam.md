---
layout: post
title: MySQL索引（8）-myisam索引结构
date: 2018-04-08
categories:
    - MySQL
comments: true
permalink: mysql-index-myisam.html
---

MyISAM引擎也使用B+Tree作为索引结构，不过与innodb不同的是**叶节点的data域存放的是数据记录的地址。**

下图是一个MyISAM表的主索引（Primary key）示例，设Col1为主键，可以看出MyISAM的索引文件仅仅保存数据记录的地址。

![](/assets/images/posts/mysql-index/mysql-index-7.png)

在MyISAM中，主索引和辅索引（Secondary key）在结构上没有任何区别，只是主索引要求key是唯一的，而辅索引的key可以重复。如果我们在Col2上建立一个辅索引，则此索引的结构如下图所示：

![](/assets/images/posts/mysql-index/mysql-index-8.png)

同样也是一颗B+Tree，data域保存数据记录的地址。
因此，MyISAM中索引检索的算法为首先按照B+Tree搜索算法搜索索引，如果指定的Key存在，则取出其data域的值，然后以data域的值为地址，读取相应数据记录。

MyISAM的索引方式也叫做“非聚集”的，之所以这么称呼是为了与InnoDB的聚集索引区分

