---
layout: post
title: nginx重定向时增加参数
date: 2019-11-14
categories:
    - nginx
comments: true
permalink: nginx-proxy-add-parameter.html
---

网上找的例子
```
server {
	listen       80;
	server_name  localhost;
	
	set $token ""; # declar token is ""(empty str) for original request without args,because $is_args concat any var will be `?`
	# 这里说默认是?，实际测试需要用set $token "?"

	if ($is_args) { # if the request has args update token to "&"
		set $token "&";
	}

	location /test {
		set $args "${args}${token}key1=val1&key2=val2"; # update original append custom params with $token
		# if no args $is_args is empty str,else it's "?"
		rewrite /api/(.*) /$1 break;
        proxy_pass http://api-server;
	}
}
```