---
layout: post
title: Nginx自定义请求头字段
date: 2019-04-10
categories:
    - nginx
comments: true
permalink: nginx-proxy-set-header.html
---

<pre class="line-numbers " data-line="2,3"><code class="language-shell">
location /header {
	proxy_set_header   iden    "student";
	proxy_set_header   age     "21";
	#proxy_set_header   Host    $host;
	proxy_pass http://api-server;
} 
</code></pre>
