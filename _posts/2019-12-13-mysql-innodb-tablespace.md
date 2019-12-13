---
layout: post
title: InnoDB架构-文件存储结构（part1）
date: 2019-05-31
categories:
    - MySQL
comments: true
permalink: innodb-tablespace.html
---

# 文件存储结构
![](/assets/images/posts/mysql-index/mysql-index-1.jpg)

## 表空间
表空间是Innodb存储引擎逻辑的最高层，所有的数据都存放在表空间中，默认情况下，Innodb存储引擎有一个共享表空间ibdata1,即所有数据都存放在这个表空间中内。如果启用了`innodb_file_per_table`参数，则每张表内的数据可以单独放到一个表空间内（*.ibd）。

- 系统表空间：系统表空间是InnoDB数据字典，双写缓冲区，Change Buffer和undo log的存储区 。属于一种共享表空间。
- 独占表空间：含单个InnoDB表的数据和索引 ，并存储在文件系统中自己的数据文件中。
- 常规表空间：类似于系统表空间，可以存储多个表的数据的一直共享表空间，支持Antelope和Barracuda文件格式。
- undo表空间：undo表空间包含undo log撤销记录的集合，其中包含通过主键索引事务撤销更改的最小信息。

**但请注意，只有数据、索引、和插入缓冲Bitmap放入单独表空间内，其他数据，比如回滚(undo)信息、插入缓冲检索页、系统事物信息，二次写缓冲等还是放在原来的共享表内的。**

## 段
表空间由段组成，常见的段有数据段、索引段、回滚段等。因为InnoDB存储引擎表是索引组织的，因此数据即索引，索引即数据。数据段即为B+树的叶子结点，索引段即为B+树的非叶子结点。
## 区
区是由连续页组成的空间，在任何情况下每个区的大小都为1MB。为了保证区中页的连续性，InnoDB存储引擎一次从磁盘申请4~5个区。默认情况下，InnoDB存储引擎页的大小为16KB，一个区中一共64个连续的区。

**在每个段开始时，先有32个页大小的碎片页（fragment page）来存放数据，当这些页使用完之后才是64个连续页的申请。这样可以避免小表浪费空间**
## 页
页是InnoDB磁盘管理的最小单位。在InnoDB存储引擎中，默认每个页的大小为16KB。可以通过参数`innodb_page_size`将页的大小设置为4K，8K，16K。若设置完成，则所有表中页的大小都固定，不可以对其再次修改。除非通过mysqldump导入和导出操作来产生新的库。

InnoDB存储引擎中，常见的页类型有：数据页，undo页，系统页，事务数据页，插入缓冲位图页，插入缓冲空闲列表页等。
## 行
InnoDB存储引擎是面向列的，数据按行存放，每页中最多存放16KB/2-200行数据

表的行格式决定了其行的物理存储方式，进而会影响查询和DML操作的性能。

##  总结一下
- 一个表的数据页是通过链表连在一起的
- 数据是以行为单位一行一行的存放在磁盘上的块中
- 在访问数据时，一次从磁盘中读出或者写入至少一个完整的页




# 参考资料

《MySQL技术内幕 InnoDB存储引擎第二版》

http://blog.codinglabs.org/articles/theory-of-mysql-index.html

https://mp.weixin.qq.com/s/R1zhVWNFtrpCSzM6fPmbBg

https://mp.weixin.qq.com/s/aH87AiBmwCtSf6z9JBc4uQ

https://www.jianshu.com/p/d4cc0ea9d097