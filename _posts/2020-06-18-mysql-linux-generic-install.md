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

1.MySQL官网下载Linux Generic版本 `wget https://cdn.mysql.com//Downloads/MySQL-8.0/mysql-8.0.21-linux-glibc2.12-x86_64.tar.xz`

2.解压`tar -xvf mysql-8.0.21-linux-glibc2.12-x86_64.tar`

3.创建/etc/my.cnf文件

```
[mysqld]
port=3306
basedir=/usr/local/mysql/mysql-8.0.21
datadir=/usr/data/mysql/data
pid-file=/usr/local/mysql/mysql-8.0.21/mysqld.pid
log-error=/usr/local/mysql/mysql-8.0.21/mysqld.err

user=root

max_connections=151

symbolic-links=0

lower_case_table_names = 1

character-set-server=utf8
 
collation-server=utf8_general_ci

bind-address = 0.0.0.0

socket=/usr/local/mysql/mysql-8.0.21/mysql.sock

[client]
port=3306
socket=/usr/local/mysql/mysql-8.0.21/mysql.sock

default-character-set=utf8
```

4.进入 `support-files/`  目录修改mysql.server  shell文件开头的basedir、datadir变量，如果指定，默认basedir是/usr/local/mysql,datadir是/usr/data/mysql/data

```
basedir=/usr/local/mysql/mysql-8.0.21
datadir=/usr/data/mysql/data
```

5.添加环境变量`vim /etc/profile`

```
MYSQL_HOME=/usr/local/mysql/mysql-8.0.21

PATH=$PATH:$MYSQL_HOME/bin
```

6.初始化数据库，得到初始随机密码（控制台没有的话可以在mysqld.err中找到）

```
./bin/mysqld --user=root --basedir=/usr/local/mysql/mysql-8.0.21 --datadir=/usr/data/mysql/data --initialize --lower-case-table-names=1
```

7.开启MySQL服务： `./support-files/mysql.server start`

8.以初始密码登陆： `mysql -u root -p` ，登陆后修改初始密码`ALTER USER 'root'@'localhost' IDENTIFIED BY '新密码';`，这里可能因为缺少mysql.sock报错，重启mydql即可

9.编辑一个Linux 服务单元文件 `/usr/lib/systemd/systemmysqld.service`，用来控制MySQL的重启和关闭

```
[Unit]
Description=MySQL Server 8.0.21
Documentation=
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/usr/local/mysql/mysql-8.0.21/mysqld.pid
ExecStart=/usr/local/mysql/mysql-8.0.21/support-files/mysql.server start
ExecReload=/usr/local/mysql/mysql-8.0.21/support-files/mysql.server restart
ExecStop=/usr/local/mysql/mysql-8.0.21/support-files/mysql.server  stop

[Install]
WantedBy=multi-user.target
```

10.设置开机自启动 `systemctl enable mysqld`



要在好几台机器上安装，写了个很low的脚本

```
#!/bin/sh

wget https://cdn.mysql.com//Downloads/MySQL-8.0/mysql-8.0.21-linux-glibc2.12-x86_64.tar.xz

tar -xvf mysql-8.0.21-linux-glibc2.12-x86_64.tar.xz -C /usr/local
mv /usr/local/mysql-8.0.21-linux-glibc2.12-x86_64 /usr/local/mysql

mkdir -p /usr/data/mysql/data

rm -rf /etc/my.cnf
touch /etc/my.cnf

echo '[mysqld]' >> /etc/my.cnf

echo 'port=3306' >> /etc/my.cnf
echo 'basedir=/usr/local/mysql' >> /etc/my.cnf
echo 'datadir=/usr/local/mysql/data' >> /etc/my.cnf
echo 'pid-file=/usr/local/mysql/mysqld.pid' >> /etc/my.cnf
echo 'log-error=/usr/local/mysql/mysqld.err' >> /etc/my.cnf
echo '' >> /etc/my.cnf
echo 'user=root' >> /etc/my.cnf
echo '' >> /etc/my.cnf
echo 'innodb_file_per_table=1' >> /etc/my.cnf
echo 'max_connections=100' >> /etc/my.cnf
echo '' >> /etc/my.cnf
echo 'symbolic-links=0' >> /etc/my.cnf
echo '' >> /etc/my.cnf
echo 'lower_case_table_names = 1' >> /etc/my.cnf
echo '' >> /etc/my.cnf
echo 'character-set-server=utf8' >> /etc/my.cnf
echo '' >> /etc/my.cnf
echo 'collation-server=utf8_general_ci' >> /etc/my.cnf
echo '' >> /etc/my.cnf
echo 'bind-address = 0.0.0.0' >> /etc/my.cnf
echo ''
echo 'socket=/usr/local/mysql/mysql.sock' >> /etc/my.cnf
echo '' >> /etc/my.cnf
echo '[client]' >> /etc/my.cnf
echo 'port=3306' >> /etc/my.cnf
echo 'socket=/usr/local/mysql/mysql.sock' >> /etc/my.cnf
echo '' >> /etc/my.cnf
echo 'default-character-set=utf8' >> /etc/my.cnf



echo 'MYSQL_HOME=/usr/local/mysql' >> /etc/profile
echo 'PATH=$PATH:$MYSQL_HOME/bin' >> /etc/profile

/usr/local/mysql/bin/mysqld --user=root --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data --initialize --lower-case-table-names=1

cat /usr/local/mysql/mysqld.err | grep root


rm -rf /usr/lib/systemd/systemmysqld.service
touch /usr/lib/systemd/systemmysqld.service
echo '[Unit]' >> /usr/lib/systemd/systemmysqld.service
echo 'Description=MySQL Server 8.0.21' >> /usr/lib/systemd/systemmysqld.service
echo 'Documentation=' >> /usr/lib/systemd/systemmysqld.service
echo 'After=network-online.target remote-fs.target nss-lookup.target' >> /usr/lib/systemd/systemmysqld.service
echo 'Wants=network-online.target' >> /usr/lib/systemd/systemmysqld.service
echo '' >> /usr/lib/systemd/systemmysqld.service
echo '[Service]' >> /usr/lib/systemd/systemmysqld.service
echo 'Type=forking' >> /usr/lib/systemd/systemmysqld.service
echo 'PIDFile=/usr/local/mysql/mysqld.pid' >> /usr/lib/systemd/systemmysqld.service
echo 'ExecStart=/usr/local/mysql/support-files/mysql.server start' >> /usr/lib/systemd/systemmysqld.service
echo 'ExecReload=/usr/local/mysql/support-files/mysql.server restart' >> /usr/lib/systemd/systemmysqld.service
echo 'ExecStop=/usr/local/mysql/support-files/mysql.server  stop' >> /usr/lib/systemd/systemmysqld.service
echo '' >> /usr/lib/systemd/systemmysqld.service
echo '[Install]' >> /usr/lib/systemd/systemmysqld.service
echo 'WantedBy=multi-user.target' >> /usr/lib/systemd/systemmysqld.service

systemctl enable mysqld
```

