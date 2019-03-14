---
layout: post
title: http跳转到https的方法
date: 2019-03-14
categories:
    - Nginx
comments: true
permalink: http-to-https.html
---

# rewrite

```
    server {  
        listen  192.168.1.111:80;  
        server_name test.com;  
          
        rewrite ^(.*)$  https://$host$1 permanent;  
    }  
```

# 497错误码

解释：当此虚拟站点只允许https访问时，当用http访问时nginx会报出497错误码

```
497 - normal request was sent to HTTPS  
```

```
    server {  
        listen       192.168.1.11:443;  #ssl端口  
        listen       192.168.1.11:80;   #用户习惯用http访问，加上80，后面通过497状态码让它自动跳到443端口  
        server_name  test.com;  
        #为一个server{......}开启ssl支持  
        ssl                  on;  
        #指定PEM格式的证书文件   
        ssl_certificate      /etc/nginx/test.pem;   
        #指定PEM格式的私钥文件  
        ssl_certificate_key  /etc/nginx/test.key;  
          
        #让http请求重定向到https请求   
        error_page 497  https://$host$uri?$args;  
    }  
```

# index刷新

```
    <html>  
    <meta http-equiv="refresh" content="0;url=https://test.com/">  
    </html>  
```