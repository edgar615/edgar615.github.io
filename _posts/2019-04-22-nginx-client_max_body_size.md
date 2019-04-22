---
layout: post
title: nginx设置文件上传大小
date: 2019-04-22
categories:
    - nginx
comments: true
permalink: client-max-body-size.html
---

在公司搭了一个gitlab，使用Nginx做反向代理，在push的时候遇到错误

```
error: RPC failed; HTTP 413 curl 22 The requested URL returned error: 413 Request Entity Too Large
```
后来发现是nginx默认设置问题
在http模块下，增加配置
```
client_max_body_size 8m;#默认1m
```
