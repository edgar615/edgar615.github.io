---
layout: post
title: Nginx使用Skywalking进行调用链跟踪
date: 2020-09-02
categories:
    - nginx
	- Skywalking
comments: true
permalink: ngnix-skywalking.html
---

Nginx使用Skywalking监控需要使用Openresty，Openresty的安装参考

> https://edgar615.github.io/openresty-install.html

下载lua脚本

```
wget https://www.apache.org/dyn/closer.cgi/skywalking/nginx-lua/0.3.0/skywalking-nginx-lua-0.3.0-src.tgz
```

配置

```
http {
    lua_package_path "/server/nginx/lua/lib/?.lua;;";
    # Buffer represents the register inform and the queue of the finished segment
    lua_shared_dict tracing_buffer 100m;

    # Init is the timer setter and keeper
    # Setup an infinite loop timer to do register and trace report.
    init_worker_by_lua_block {
        local metadata_buffer = ngx.shared.tracing_buffer

        metadata_buffer:set('serviceName', 'Nginx-Service')
        -- Instance means the number of Nginx deloyment, does not mean the worker instances
        metadata_buffer:set('serviceInstanceName', 'Nginx-Service-1')

        require("skywalking.client"):startBackendTimer("http://192.168.159.131:12800")
    }

	...

    upstream test-server {
      server 192.168.159.1:9001;
    }

    #gzip  on;

    server {
	...
        location /students/ {
            rewrite_by_lua_block {
                ------------------------------------------------------
                -- NOTICE, this should be changed manually
                -- This variable represents the upstream logic address
                -- Please set them as service logic name or DNS name
                --
                -- Currently, we can not have the upstream real network address
                ------------------------------------------------------
                require("skywalking.tracer"):start("upstream service")
                -- If you want correlation custom data to the downstream service
                -- require("skywalking.tracer"):start("upstream service", {custom = "custom_value"})
            }
           proxy_pass http://test-server;
           body_filter_by_lua_block {
                if ngx.arg[2] then
                    require("skywalking.tracer"):finish()
                end
            }

            log_by_lua_block {
                require("skywalking.tracer"):prepareForReport()
            }
        }

    }

```

**注意`require("skywalking.client"):startBackendTimer("http://192.168.159.131:12800")`**的端口是12800，使用rest接口上报

参考资料


https://github.com/apache/skywalking-nginx-lua/