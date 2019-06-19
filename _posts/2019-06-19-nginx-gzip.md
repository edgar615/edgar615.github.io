---
layout: post
title: nginx gzip
date: 2019-06-19
categories:
    - nginx
comments: true
permalink: nginx-gzip.html
---

直接上配置
<pre class="line-numbers"><code>
    #开启和关闭gzip模式
    gzip on|off;
    
    #gizp压缩起点，文件大于1k才进行压缩
    gzip_min_length 1k;
    
    # gzip 压缩级别，1-9，数字越大压缩的越好，也越占用CPU时间
    gzip_comp_level 1;
    
    # 进行压缩的文件类型。
    gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript ;
    
    # 是否在http header中添加Vary: Accept-Encoding，建议开启
    gzip_vary on;

    # 设置压缩所需要的缓冲区大小，以4k为单位，如果文件为7k则申请2*4k的缓冲区 
    gzip_buffers 2 4k;

    # 设置gzip压缩针对的HTTP协议版本
    gzip_http_version 1.1;
</code></pre>