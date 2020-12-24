---
layout: post
title: OpenSSL升级
date: 2020-06-22
categories:
    - linux
comments: true
permalink: openssl-upgrade.html
---

近期升级了一下openssl的版本

# 1. 安装依赖

```
yum group install 'Development Tools'
yum install perl-core zlib-devel -y
```

# 2. 下载openssl并解压

```
wget https://www.openssl.org/source/openssl-1.1.1c.tar.gz
tar -xf openssl-1.1.1c.tar.gz
```

# 3. 安装

```
cd openssl-1.1.1c
./config --prefix=/usr/local/ssl --openssldir=/usr/local/ssl shared zlib
make
make test
make install
```

# 4. 配置
```
cd /etc/ld.so.conf.d/
```
创建`openssl-1.1.1c.conf`文件，写入

```
/usr/local/ssl/lib
```

reload the dynamic link

```
ldconfig -v
```

# 5. Configure OpenSSL Binary

```
mv /bin/openssl /bin/openssl.backup
```

创建`/etc/profile.d/openssl.sh`文件，写入

```
#Set OPENSSL_PATH
OPENSSL_PATH="/usr/local/ssl/bin"
export OPENSSL_PATH
PATH=$PATH:$OPENSSL_PATH
export PATH
```

设置为可执行

```
chmod +x /etc/profile.d/openssl.sh
source /etc/profile.d/openssl.sh
```

# 6. 检查版本

```
openssl version -a
```
