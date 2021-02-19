---
layout: post
title: MySQL安装与使用（3）- Ubuntu安装MySQL8
date: 2018-03-03
categories:
    - MySQL
comments: true
permalink: mysql-ubuntu-install.html
---

下载MySQL配置文件

```
https://dev.mysql.com/downloads/repo/apt/
```

安装mysql-apt-config_0.8.16-1_all.deb

```
# dpkg -i mysql-apt-config_0.8.16-1_all.deb
Selecting previously unselected package mysql-apt-config.
(Reading database ... 71034 files and directories currently installed.)
Preparing to unpack mysql-apt-config_0.8.16-1_all.deb ...
Unpacking mysql-apt-config (0.8.16-1) ...
Setting up mysql-apt-config (0.8.16-1) ...
Warning: apt-key should not be used in scripts (called from postinst maintainerscript of the package mysql-apt-config)
OK
```

安装MySQL

```
# apt-get update
# apt install mysql-server
```

