---
layout: post
title: ShardingSphere（4）- 数据加密
date: 2020-11-13
categories:
    - 分库分表
comments: true
permalink: shardingsphere-encrypt.html
---

**数据脱敏**，是指对某些敏感信息通过脱敏规则进行数据转换，从而实现敏感隐私数据的可靠保护。在日常开发过程中，数据安全一直是一个非常重要和敏感的话题。相较传统的私有化部署方案，互联网应用对数据安全的要求更高，所涉及的范围也更广。根据不同行业和业务场景的属性，不同系统的敏感信息可能有所不同，但诸如身份证号、手机号、卡号、用户姓名、账号密码等个人信息一般都需要进行脱敏处理。

# 1. 实现原理

对于一些敏感数据而言，我们显然应该直接以密文的形式将加密之后的数据进行存储，防止有任何一种途径能够从数据库中获取这些数据明文。 在这类敏感数据中，最典型的就是用户密码，我们通常会采用 MD5 等不可逆的加密算法对其进行加密，而使用这些数据的方法也只是依赖于它的密文形式，不会涉及对明文的直接处理。

但对于用户姓名、手机号等信息，由于统计分析等方面的需要，显然我们不能直接采用不可逆的加密算法对其进行加密，还需要将明文信息进行处理**。**一种常见的处理方式是将一个字段用两列来进行保存，一列保存明文，一列保存密文，这就是第二种情况。

显然，我们可以将第一种情况看作是第二种情况的特例。也就是说，在第一种情况中没有明文列，只有密文列。

ShardingSphere 同样基于这两种情况进行了抽象，它将这里的明文列命名为 plainColumn，而将密文列命名为 cipherColumn。其中 plainColumn 属于选填，而 cipherColumn 则是必填。同时，ShardingSphere 还提出了一个逻辑列 logicColumn 的概念，该列代表一种虚拟列，只面向开发人员进行编程使用：

![](/assets/images/posts/shardingsphere-encrypt/shardingsphere-encrypt-1.png)

举例说明，假如数据库里有一张表叫做 `t_user`，这张表里实际有两个字段 `pwd_plain`，用于存放明文数据、`pwd_cipher`，用于存放密文数据，同时定义 logicColumn 为 `pwd`。 那么，用户在编写 SQL 时应该面向 logicColumn 进行编写，即 `INSERT INTO t_user SET pwd = '123'`。 Apache ShardingSphere 接收到该 SQL，通过用户提供的加密配置，发现 `pwd` 是 logicColumn，于是便对逻辑列及其对应的明文数据进行加密处理。 **Apache ShardingSphere 将面向用户的逻辑列与面向底层数据库的明文列和密文列进行了列名以及数据的加密映射转换。** 如下图所示：

![](/assets/images/posts/shardingsphere-encrypt/shardingsphere-encrypt-2.png)

即依据用户提供的加密规则，将用户 SQL 与底层数据表结构割裂开来，使得用户的 SQL 编写不再依赖于真实的数据库表结构。 而用户与底层数据库之间的衔接、映射、转换交由 Apache ShardingSphere 进行处理。

下方图片展示了使用加密模块进行增删改查时，其中的处理流程和转换逻辑，如下图所示。

![](/assets/images/posts/shardingsphere-encrypt/shardingsphere-encrypt-3.png)

## 2.  使用

准备一个表

```
create table encrypt_user 
(
   user_id              bigint not null comment 'id',
   username             varchar(60) comment '用户名密文',
   username_plain       varchar(16) comment '用户名明文',
   password             varchar(60) comment '密码密文',
   primary key (user_id)
);
```

我们的实体类不定义user_plain字段

```
public class EncryptUser {

    private Long userId;

    private String username;
    
    private String password;
    
}
```

配置数据源

```
spring:
  shardingsphere:
    datasource:
      names: dsencrypt
      dsencrypt:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://192.168.159.133:3306/ds0
        username: root
        password: 123456
```

定义 name_encryptor 和 pwd_encryptor 两个加密器，分别用于对 user_name 列和 pwd 列进行加解密

```
spring:
  shardingsphere:
    encrypt:
      encryptors:
        name_encryptor:
          type: aes
          props:
            aes:
              key:
                value: 123456
        pwd_encryptor:
          type: md5
```

脱敏表设置

```
spring:
  shardingsphere:
    encrypt:
      tables:
        encrypt_user: # 表名
          columns:
            username:
              plain-column: username_plain # 明文列
              cipher-column: username #加密列
              encryptor: name_encryptor #加密算法
            password:
              cipher-column: password #加密列
              encryptor: pwd_encryptor #加密算法
```

执行insert后，我们可以看到encrypt_user中的username和password已经加密了

```
mysql> select * from encrypt_user;
+---------+--------------------------+----------------+----------------------------------+
| user_id | username                 | username_plain | password                         |
+---------+--------------------------+----------------+----------------------------------+
|      10 | SxshTO7kdGEixk7Ls7pCIQ== | edgar10        | e10adc3949ba59abbe56e057f20f883e |
|      11 | WHsI7HCTDcUUM3OO8f2yNg== | edgar11        | e10adc3949ba59abbe56e057f20f883e |
|      12 | P831ywziyuvCp4QFkjm0/Q== | edgar12        | e10adc3949ba59abbe56e057f20f883e |
|      13 | YhZoUOMer77RtKvIP+/F8Q== | edgar13        | e10adc3949ba59abbe56e057f20f883e |
|      14 | mMw/nw9RcwenpmTyTbgeIA== | edgar14        | e10adc3949ba59abbe56e057f20f883e |
|      15 | CkY4FWCr40/F0l+r7O2MVg== | edgar15        | e10adc3949ba59abbe56e057f20f883e |
|      16 | 6DOAvVuEgTYCQuBi8htZyw== | edgar16        | e10adc3949ba59abbe56e057f20f883e |
|      17 | B19KS700PPbL2qhfvfqOHQ== | edgar17        | e10adc3949ba59abbe56e057f20f883e |
|      18 | yo/uYaAXobFdu+k8N799gw== | edgar18        | e10adc3949ba59abbe56e057f20f883e |
|      19 | /oCU0KgTXa++rMGNT0ZkTQ== | edgar19        | e10adc3949ba59abbe56e057f20f883e |
+---------+--------------------------+----------------+----------------------------------+
10 rows in set (0.00 sec)
```

执行查询后，我们可以看到username返回的是明文

```
[User{userId=11, username=edgar11, password=e10adc3949ba59abbe56e057f20f883e}]
```

ShardingSphere 提供了一个属性开关，当底层数据库表里同时存储了明文和密文数据后，该属性开关可以决定是直接查询数据库表里的明文数据进行返回，还是查询密文数据并进行解密之后再返回：

```
spring:
  shardingsphere:
    props:
      query:
        with:
          cipher:
            column: true
```

> 测试没起作用，后续研究

# 4. 参考资料

《ShardingSphere 核心原理精讲 》

https://shardingsphere.apache.org/document/current/cn/features/encrypt/principle/