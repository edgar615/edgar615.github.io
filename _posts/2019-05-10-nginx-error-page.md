---
layout: post
title: nginx设置错误页
date: 2019-05-10
categories:
    - nginx
comments: true
permalink: nginx-error-page.html
---

直接上配置
<pre class="line-numbers"><code>
error_page 401 /401.html;
error_page 403 /403.html;
error_page 404 /404.html;
error_page 500 502 503 504 /500.html;
location = /401.html {
	root   /alidata/nginx/content/error;
}
location = /403.html {
	root   /alidata/nginx/content/error;
}
location = /404.html {
	root   /alidata/nginx/content/error;
}
location = /500.html {
	root   /alidata/nginx/content/error;
}
</code></pre>