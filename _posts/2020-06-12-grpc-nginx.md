---
layout: post
title: GRPC（12）- 使用Nginx反向代理
date: 2020-06-12
categories:
    - rpc
comments: true
permalink: grpc-nginx.html
---

Nginx 在 1.13.10 中，新增了对gRPC的原生支持。

有了对 gRPC 的支持，NGINX 就可以代理 gRPC TCP 连接，还可以终止、检查和跟踪 gRPC 的方法调用。你可以：

- 发布 gRPC 服务，然后使用 NGINX 应用 HTTP/2 TLS 加密、速率限制、基于 IP 的访问控制列表和日志记录。
- 通过单个端点发布多个 gRPC 服务，使用 NGINX 检查并跟踪每个内部服务的调用。
- 对一组 gRPC 服务进行负载均衡，可以使用轮询算法、最少连接数原则或其他方式在集群上分发流量。

# 1. 暴露简单的 gRPC 服务

首先，在客户端和服务器端之间插入 NGINX，NGINX 为服务器端的应用程序提供了一个稳定可靠的网关。

NGINX 使用 HTTP 服务器监听 gRPC 流量，并使用 grpc_pass 指令代理流量。 为 NGINX 创建以下代理配置，在端口 80 上侦听未加密的 gRPC 流量并将请求转发到端口 50051 上的服务器：

![](/assets/images/posts/grpc-nginx/grpc-nginx-1.png)

```
http {
  log_format main '$remote_addr - $remote_user [$time_local] "$request" '
           '$status $body_bytes_sent "$http_referer" '
           '"$http_user_agent"';

  server {
    listen 80 http2;

    access_log logs/access.log main;

    location / {
      # Replace localhost:50051 with the address and port of your gRPC server
      # The 'grpc://' prefix is optional; unencrypted gRPC is the default   
      grpc_pass grpc://localhost:50051;
    }
  }
}
```

指令grpc_pass用来指定代理的gRPC服务器地址，前缀协议有两种：

- grpc://：与gRPC服务器端交互是以明文的方式
- grpcs://：与gRPC服务器端交互式以TLS加密方式

gRPC服务器地址的前缀“grpc://”是可以忽略，默认就是明文交互方式。

此示例里nginx以明文的方式在80端口发布gRPC，其中代理的gRPC在后端也是以明文的方式交互。

**注意：Nginx是不支持在明文的端口上同时支持http1和http2的。如果要支持这两种的http协议，需要设置为不同的端口。**

观察请求日志

```
192.168.159.1 - - [11/Jan/2021:07:16:12 +0000] "POST /com.github.edgar615.grpc.Greeter/SayHello HTTP/2.0" 200 18 "-" "grpc-java-netty/1.30.2" "-"
```

# 2. 使用 TLS 加密发布 gRPC 服务

上面的示例使用未加密的 HTTP/2（明文）进行通信。这对测试和部署来说非常简单，但生产环境为了安全需要加密，你可以使用 NGINX 来添加这个加密层。

![](/assets/images/posts/grpc-nginx/grpc-nginx-2.png)

配置示例如下：

```
server {
  listen 1443 ssl http2;

  ssl_certificate ssl/cert.pem;
  ssl_certificate_key ssl/key.pem;

  location / {
      grpc_pass grpc://localhost:50051;
  }
}
```

示例里在nginx层给gRPC服务添加了ssl证书，而内部代理到gRPC服务器仍然是使用明文的交互方式，也就是在Nginx层，做到了SSL offloading。

gRPC客户端也是需要TLS加密。如果是使用自签名证书等未经信任的证书，客户端都需要禁用证书检查。在部署到生产环境时，需要将自签名证书换成由可信任证书机构发布的证书，客户端也需要配置成信任该证书。

# 3. 反向代理加密的 gRPC 服务

如果Nginx内部代理的gRPC也需要以加密的方式交互，这种情况就需要把明文代理协议grpc://替换为grpcs://。这首先要gRPC服务器是以加密的方式发布服务的。Nginx层修改如下：

```
grpc_pass grpcs://localhost:50051;
```

# 4. 路由 gRPC 流量

如果后端有多个gRPC服务端，其中每个服务端都是提供不同的gRPC服务。这种情况可以使用一个nginx接收客户端请求，然后根据不同的路径分发路由到指定的gRPC服务器。使用location区分：

```
location /helloworld.Greeter {
  grpc_pass grpc://192.168.20.11:50051;
}

location /helloworld.Dispatcher {
  grpc_pass grpc://192.168.20.21:50052;
}

location / {
  root html;
  index index.html index.htm;
}
```

`/helloworld.Greeter`对应了包名+接口名

# 5. 对 gRPC 流量进行负载均衡

在后端有多个gRPC服务器，它们都是同一个gRPC服务，这种情况可以结合nginx的upstream可以对gRPC的请求做负载均衡。

```
upstream grpcservers {
  server 192.168.20.21:50051;
  server 192.168.20.22:50052;
}

server {
  listen 1443 ssl http2;

  ssl_certificate   ssl/certificate.pem;
  ssl_certificate_key ssl/key.pem;

  location /helloworld.Greeter {
    grpc_pass grpc://grpcservers;
    error_page 502 = /error502grpc;
  }

  location = /error502grpc {
    internal;
    default_type application/grpc;
    add_header grpc-status 14;
    add_header grpc-message "unavailable";
    return 204;
  }
}
```

其中upstream指定定义了统一gRPC服务的服务器组。在grpc_pass指定的gRPC服务器地址使用upstream定义的服务器组。

NGINX 支持多种负载均衡算法，其内置的健康检测机制可以检测到无法及时响应或发生错误的服务器，并把它们移除。如果没有可用的服务器，就会返回 `/error502grpc` 指定的错误消息。

# 6. 其他指令

ngx_http_grpc_module模块提供了很多指令用于配置grpc的反向代理，如**grpc_connect_timeout**，**grpc_set_header** 等，具体用法参考官方文档

> http://nginx.org/en/docs/http/ngx_http_grpc_module.html

# 7. 参考资料

https://www.nginx.com/blog/nginx-1-13-10-grpc/

http://nginx.org/en/docs/http/ngx_http_grpc_module.html