---
layout: post
title: 限流（4）- Nginx
date: 2020-04-18
categories:
    - 架构设计,nginx
comments: true
permalink: ratelimit-ngnix.html
---

nginx自带两个限流模块

- 连接数限流模块`ngx_http_limit_conn_module`
- 请求限流模块`ngx_http_limit_req_module` 漏桶实现

OpenResty提供了`lua-resty-limit-traffic`可以实现更复杂的限流

# 1. ngx_http_limit_conn_module

我们经常会遇到这种情况，服务器流量异常，负载过大等等。对于大流量恶意的攻击访问，会带来带宽的浪费，服务器压力，影响业务，往往考虑对同一个ip的连接数，并发数进行限制。

ngx_http_limit_conn_module 模块来实现该需求。该模块可以根据定义的键来限制每个键值的连接数，如同一个IP来源的连接数。并不是所有的连接都会被该模块计数，只有那些正在被处理的请求（这些请求的头信息已被完全读入）所在的连接才会被计数。

**配置示例**

可以在nginx_conf的http{}中加上如下配置实现限制：

```
#限制每个用户的并发连接数，取名one
limit_conn_zone $binary_remote_addr zone=one:10m;

#配置记录被限流后的日志级别，默认error级别
limit_conn_log_level error;
#配置被限流后返回的状态码，默认返回503
limit_conn_status 503;
```

然后在server{}里加上如下代码：(也可以设置在单独的location)

```
#限制用户并发连接数为1
limit_conn one 1;
```

- limit_conn: 配置存放key和计数器的共享区域和指定key的最大连接数，此处指定最大连接数是1，表述nginx最多同时并发处理一个请求
- limit_conn_zone 配置限流key及存放key对应信息的共享内存区域大小，此处的key是IP地址，也可以使用`$server_name`做完key来现在域名级别的最大连接数
- limit_conn_status 配置被限流后返回的状态码，默认503
-  limit_conn_log_level 配置记录被限流后的日志级别，默认error

然后我们是使用ab测试来模拟并发请求：

```
ab -n 5 -c 5 http://localhost/
```

另外刚才是配置针对单个IP的并发限制，还是可以针对域名进行并发限制，配置和客户端IP类似。

```
#http{}段配置
limit_conn_zone $ server_name zone=perserver:10m;
#server{}段配置
limit_conn perserver 1;
```

# 2. ngx_http_limit_req_module

ngx_http_limit_req_module模块可以通过定义的键值来限制请求处理的频率。特别的，可以限制来自单个IP地址的请求处理频率。限制的方法是使用了漏斗算法，每秒固定处理请求数，推迟过多请求。如果请求的频率超过了限制域配置的值，请求处理会被延迟或被丢弃，所以所有的请求都是以定义的频率被处理的。

在http{}中配置

```
#区域名称为one，大小为10m，平均处理的请求频率不能超过每秒一次。
limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
```

在server{}中配置（也可以设置在单独的location）

```
#设置每个IP桶的数量为5
limit_req zone=one burst=5 nodelay;
#limit_req zone=one burst=5;
```

上面设置定义了每个IP的请求处理只能限制在每秒1个。并且服务端可以为每个IP缓存5个请求，如果操作了5个请求，请求就会被丢弃。

- limit_req 配置限流区域、桶容量（突发容量，默认为0）、是否延迟模式（默认延迟）
- limit_req_zone 配置限流key，存放key对应信息的共享内存区域大小、固定请求速率，请求速率使用rate参数设置，支持`10r/s`和`60r/m`最终都会转换为每秒的固定请求
- limit_conn_status 配置被限流后返回的状态码，默认503
- limit_conn_log_level 配置记录被限流后的日志级别，默认error

延迟模式下，时间窗口内的请求会被暂存到桶中，并以固定平均速率处理请求

limit_req_zone/limit_conn_zone定义的内存不足，后续的请求将一直被限流，所以要根据需求设置好相应的内存大小

 limit_req_zone/limit_conn_zone的限流都是基于单nginx，假设我们接入层有多个nginx，这时我们就需要使用lua-resty-limit-traffic

# 3. lua-resty-limit-traffic

OpenResty的lua-resty-limit-traffic也提供了limit.conn和limit.req实现

配置用来存放限流用的共享字典：

```
lua_shared_dict limit_req_store 100m;
```

限流的lua代码

```
local limit_req = require "resty.limit.req"
local rate = 2 --固定平均速率 2r/s
local burst = 3 --桶容量
local error_status = 503
local nodelay = false --是否需要不延迟处理
local lim, err = limit_req.new("limit_req_store", rate, burst)
if not lim then --没定义共享字典
    ngx.exit(error_status)
end
local key = ngx.var.binary_remote_addr --IP维度的限流
--流入请求，如果请求需要被延迟则delay > 0
local delay, err = lim:incoming(key, true)
if not delay and err == "rejected" then --超出桶大小了
    ngx.exit(error_status)
end
if delay > 0 then --根据需要决定是延迟或者不延迟处理
    if nodelay then
        --直接突发处理了
    else
        ngx.sleep(delay) --延迟处理
    end
end
```

即限流逻辑再nginx access阶段被访问，如果不被限流继续后续流程；如果需要被限流要么sleep一段时间继续后续流程，要么返回相应的状态码拒绝请求。

另外在使用Nginx+Lua时也可以获取ngx.var.connections_active进行过载保护，即如果当前活跃连接数超过阈值进行限流保护。

```
if tonumber(ngx.var.connections_active) >= tonumber(limit) then
    //限流
end
```

nginx也提供了limit_rate用来对流量限速，如limit_rate 50k，表示限制下载速度为50k。

