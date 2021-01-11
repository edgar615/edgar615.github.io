---
layout: post
title: Nginx使用zipkin进行调用链跟踪
date: 2020-04-19
categories:
    - nginx
	- Zipkin
comments: true
permalink: ngnix-zipkin.html
---

下载对应版本的OpenTracing插件并解压。

```
wget https://github.com/opentracing-contrib/nginx-opentracing/releases/download/v0.10.0/linux-amd64-nginx-1.18.0-ngx_http_module.so.tgz
tar -zxvf linux-amd64-nginx-1.18.0-ngx_http_module.so.tgz
```

拷贝.so文件至Nginx的modules目录。如果不存在该目录则需要先创建。

 ```
mv ngx_http_opentracing_module.so /etc/nginx/modules
 ```

下载Zipkin插件并将其拷贝至任意工作目录。

```
wget  https://github.com/rnburn/zipkin-cpp-opentracing/releases/download/v0.5.2/linux-amd64-libzipkin_opentracing_plugin.so.gz
gunzip linux-amd64-libzipkin_opentracing_plugin.so.gz
cp linux-amd64-libzipkin_opentracing_plugin.so /usr/local/lib/libzipkin_opentracing_plugin.so
```

配置/usr/local/nginx/conf/nginx.conf文件

```
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

load_module modules/ngx_http_opentracing_module.so;

events {
    worker_connections  1024;
}


http {
    opentracing on;

    opentracing_load_tracer /usr/local/lib/libzipkin_opentracing_plugin.so /etc/zipkin/zipkin-config.json;

}

```

在**/etc/zipkin/zipkin-config.json**文件中配置Zipkin参数。

```
{
  "service_name": "nginx",
  "collector_host": "Zipkin-server-IP-address",
  "collector_port": 9411,
	"sample_rate":0.1
}
```

可以增加默认tag

```
# Enable tracing for all requests
opentracing on;

# Set additional tags that capture the value of NGINX Plus variables
opentracing_tag bytes_sent $bytes_sent;
opentracing_tag http_user_agent $http_user_agent;
opentracing_tag request_time $request_time;
opentracing_tag upstream_addr $upstream_addr;
opentracing_tag upstream_bytes_received $upstream_bytes_received;
opentracing_tag upstream_cache_status $upstream_cache_status;
opentracing_tag upstream_connect_time $upstream_connect_time;
opentracing_tag upstream_header_time $upstream_header_time;
opentracing_tag upstream_response_time $upstream_response_tim
```

修改location配置

```

    location  /students/ {
        opentracing_operation_name $uri;
        # 如果opentracing_trace_locations设置为false，zipkin中Nginx和下游test-server之间的span会断开，没有联系
        #opentracing_trace_locations off;
        # Propagate the active Span context upstream, so that the trace can be
        # continued by the backend.
        opentracing_propagate_context;
        proxy_pass http://test-server;
    }
```

参考资料

https://www.nginx.com/blog/opentracing-nginx-plus/

https://help.aliyun.com/document_detail/193866.html