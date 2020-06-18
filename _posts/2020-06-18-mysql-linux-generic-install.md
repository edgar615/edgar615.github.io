---
layout: post
title: 安装MySQL Linux Generic版本
date: 2020-06-18
categories:
    - MySQL
comments: true
permalink: mysql-linux-generic-install.html
---

内网环境要用MySQL，网上学习了下周末安装Generic版MySQL

1.MySQL官网下载Linux Generic版本

2.解压`tar -xvf mysql-8.0.20-linux-glibc2.12-x86_64.tar`

3.创建/etc/my.cnf文件

```
[mysqld]
port=3306
basedir=/usr/local/mysql/mysql-8.0.20
datadir=/usr/data/mysql/data
pid-file=/usr/local/mysql/mysql-8.0.20/mysqld.pid
log-error=/usr/local/mysql/mysql-8.0.20/mysqld.err

user=root

max_connections=151

symbolic-links=0

lower_case_table_names = 1

character-set-server=utf8
 
collation-server=utf8_general_ci

bind-address = 0.0.0.0

socket=/usr/local/mysql/mysql-8.0.20/mysql.sock

[client]
port=3306
socket=/usr/local/mysql/mysql-8.0.20/mysql.sock

default-character-set=utf8
```

4.进入 `support-files/`  目录修改mysql.server  shell文件开头的basedir、datadir变量

```
basedir=/usr/local/mysql/mysql-8.0.20
datadir=/usr/data/mysql/data
```

5.添加环境变量`vim /etc/profile`

```
MYSQL_HOME=/usr/local/mysql/mysql-8.0.20

PATH=$PATH:$MYSQL_HOME/bin
```

6.初始化数据库，得到初始随机密码（控制台没有的话可以在mysqld.err中找到）

```
./bin/mysqld --user=root --basedir=/usr/local/mysql/mysql-8.0.20 --datadir=/usr/data/mysql/data --initialize --lower-case-table-names=1
```

7.开启MySQL服务： `./support-files/mysql.server start`

8.以初始密码登陆： `mysql -u root -p` ，登陆后修改初始密码，这里可能因为缺少mysql.sock报错，重启mydql即可

9.编辑一个Linux 服务单元文件 `/usr/lib/systemd/systemmysqld.service`，用来控制MySQL的重启和关闭

```
[Unit]
Description=MySQL Server 8.0.20
Documentation=
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/usr/local/mysql/mysql-8.0.20/mysqld.pid
ExecStart=/usr/local/mysql/mysql-8.0.20/support-files/mysql.server start
ExecReload=/usr/local/mysql/mysql-8.0.20/support-files/mysql.server restart
ExecStop=/usr/local/mysql/mysql-8.0.20/support-files/mysql.server  stop

[Install]
WantedBy=multi-user.target
```

10.设置开机自启动 `systemctl enable mysqld`
