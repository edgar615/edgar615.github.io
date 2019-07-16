---
layout: post
title: openresty安装
date: 2019-07-16
categories:
    - nginx
comments: true
permalink: openresty-install.html
---



1.下载源码

```
wget https://github.com/openresty/openresty/releases/download/v1.13.6.1/openresty-1.13.6.1.tar.gz
```

2.解压

```
tar -zx -f openresty-1.13.6.1.tar.gz

cd openresty-1.13.6.1
```

3.编译

```
./configure -j2
make -j2
sudo make install

# better also add the following line to your ~/.bashrc or ~/.bash_profile file.
export PATH=/usr/local/openresty/bin:$PATH
```

```
./configure -j2会报错
./configure: error: the HTTP rewrite module requires the PCRE library.
安装pcre-devel
yum -y install pcre-devel

./configure: error: SSL modules require the OpenSSL library.
安装 openssl
yum -y install openssl openssl-devel
```



安装完后

```
PATH=/usr/local/openresty/nginx/sbin:/usr/local/openresty/bin:$PATH
export PATH
```

配置文件放在，别的地方reload各种失败

```
/usr/local/openresty/nginx/conf/nginx.conf
```



有时候出现

```
[error] open() "/usr/local/openresty/nginx/logs/nginx.pid
```

执行

```
nginx -c /usr/local/openresty/nginx/conf/nginx.conf
```



有时候出现

```
nginx: [emerg] getpwnam("nginx")
```

执行

```
/usr/sbin/groupadd -f nginx
/usr/sbin/useradd -g nginx nginx
```

