---
layout: post
title: http��ת��https�ķ���
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

# 497������

���ͣ���������վ��ֻ����https����ʱ������http����ʱnginx�ᱨ��497������

```
497 - normal request was sent to HTTPS  
```

```
    server {  
        listen       192.168.1.11:443;  #ssl�˿�  
        listen       192.168.1.11:80;   #�û�ϰ����http���ʣ�����80������ͨ��497״̬�������Զ�����443�˿�  
        server_name  test.com;  
        #Ϊһ��server{......}����ssl֧��  
        ssl                  on;  
        #ָ��PEM��ʽ��֤���ļ�   
        ssl_certificate      /etc/nginx/test.pem;   
        #ָ��PEM��ʽ��˽Կ�ļ�  
        ssl_certificate_key  /etc/nginx/test.key;  
          
        #��http�����ض���https����   
        error_page 497  https://$host$uri?$args;  
    }  
```

# indexˢ��

```
    <html>  
    <meta http-equiv="refresh" content="0;url=https://test.com/">  
    </html>  
```