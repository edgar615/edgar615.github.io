---
layout: post
title: nginx配置虚拟主机
date: 2019-03-14
categories:
    - nginx
comments: true
permalink: nginx-vhost.html
---

# 通配符

```
    server {		
        listen       80 default_server;
        server_name  "~^(?<name>\w+)\.example\.com$";
		root /alidata/nginx/content/$name;
		#access_log /alidata/nginx/log/$name/access.log; #不起作用
		#error_log  /alidata/nginx/log/$name/error.log; #不起作用

        # Load configuration files for the default server block.
        #include /etc/nginx/default.d/*.conf;

        location / {
		   index  index.html;
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
```

# 完整复杂的方法

http模块中增加一行

<pre class="line-numbers"><code class="language-nginx">
include /alidata/nginx/vhost/*.conf;
</code></pre>

在vhost目录下增加www.conf, admin.conf shop.conf分别设置

www.conf

<pre class="line-numbers"><code class="language-nginx">
server {
	listen       80 default_server;
	server_name  www.example.com;
	root         /alidata/nginx/content;
	location / {
	   root /alidata/nginx/content;
	   index  index.html;
	}

	error_page 404 /404.html;
		location = /40x.html {
	}
	
	error_page 500 502 503 504 /50x.html;
		location = /50x.html {
	}
}
</code></pre>

admin.conf

<pre class="line-numbers"><code class="language-nginx">
server {
	listen       80;
	server_name  admin.tabaosmart.com;
	root         /alidata/nginx/admin;

	location / {
	   root /alidata/nginx/admin;
	   index  index.html;
	}
	
	error_page 404 /404.html;
		location = /40x.html {
	}
	
	error_page 500 502 503 504 /50x.html;
		location = /50x.html {
	}
}
</code></pre>

shop.conf

<pre class="line-numbers"><code class="language-nginx">
server {
	listen       80;
	server_name  shop.tabaosmart.com;
	root         /alidata/nginx/shop;

	location / {
	   root /alidata/nginx/shop;
	   index  index.html;
	}
	
	error_page 404 /404.html;
		location = /40x.html {
	}
	
	error_page 500 502 503 504 /50x.html;
		location = /50x.html {
	}
}
</code></pre>

备注：不要忘记设置一个default_server，不然nginx找了一圈都没有匹配后，会直接路由到它找到的第一个配置的虚拟主机上.

> 网上看到是按虚拟主机的conf文件名字的字母表顺序，没有实测出这种规律，有点像是按我创建vhost的顺序

# 参考资料

http://nginx.org/en/docs/http/server_names.html