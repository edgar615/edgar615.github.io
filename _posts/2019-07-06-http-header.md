---
layout: post
title: Http常用请求头和响应头
date: 2019-07-06
categories:
    - http
comments: true
permalink: http-header.html
---

# 请求头
## Accept
用来告知客户端可以处理的内容类型，这种内容类型用MIME类型来表示。借助内容协商机制, 服务器可以从诸多备选项中选择一项进行应用，并使用 Content-Type 应答头通知客户端它的选择。浏览器会基于请求的上下文来为这个请求头设置合适的值，比如获取一个CSS层叠样式表时值与获取图片、视频或脚本文件时的值是不同的。

```
Accept: text/html
Accept: image/*
Accept: text/html, application/xhtml+xml, application/xml;q=0.9, */*;q=0.8
```

- `<MIME_type>/<MIME_subtype>`   单一精确的 MIME 类型, 例如text/html.
- `<MIME_type>/*`一类 MIME 类型, 但是没有指明子类。 image/* 可以用来指代 image/png, image/svg, image/gif 以及任何其他的图片类型。
- `*/*`   任意类型的 MIME 类型
- `;q= (q因子权重)`   值代表优先顺序，用相对质量价值表示，又称作权重。

## Accept-Charset
用来告知（服务器）客户端可以处理的字符集类型。 借助内容协商机制，服务器可以从诸多备选项中选择一项进行应用， 并使用Content-Type 应答头通知客户端它的选择。浏览器通常不会设置此项值，因为每种内容类型的默认值通常都是正确的，但是发送它会更有利于识别。

如果服务器不能提供任何可以匹配的字符集的版本，那么理论上来说应该返回一个 406 （Not Acceptable，不被接受）的错误码。但是为了更好的用户体验，这种方法很少采用，取而代之的是将其忽略。

```
Accept-Charset: iso-8859-1
Accept-Charset: utf-8, iso-8859-1;q=0.5
Accept-Charset: utf-8, iso-8859-1;q=0.5, *;q=0.1
```


- `<charset>`    诸如 utf-8 或 iso-8859-15的字符集。
- `*`  在这个消息头中未提及的任意其他字符集；'*' 用来表示通配符。
- `;q= (q-factor weighting)`  值代表优先顺序，用相对质量价值表示，又称为权重。 

## Accept-Encoding
会将客户端能够理解的内容编码方式(通常是某种压缩算法)进行通知。通过内容协商的方式，服务端会选择一个客户端提议的方式，使用并在响应报文首部 `Content-Encoding` 中通知客户端该选择。

即使客户端和服务器都支持相同的压缩算法，在 identity 指令可以被接受的情况下，服务器也可以选择对响应主体不进行压缩。导致这种情况出现的两种常见的情形是：

    要发送的数据已经经过压缩，再次进行压缩不会导致被传输的数据量更小。一些图像格式的文件会存在这种情况；
    服务器超载，无法承受压缩需求导致的计算开销。通常，如果服务器使用超过80%的计算能力，微软建议不要压缩。

```
Accept-Encoding: gzip
Accept-Encoding: gzip, compress, br
Accept-Encoding: br;q=1.0, gzip;q=0.8, *;q=0.1
```
- `gzip`  表示采用 Lempel-Ziv coding (LZ77) 压缩算法，以及32位CRC校验的编码方式。
- `compress`  采用 Lempel-Ziv-Welch (LZW) 压缩算法。
- `deflate` 采用 zlib 结构和 deflate 压缩算法。
- `br` 表示采用 Brotli 算法的编码方式。
- `identity`  用于指代自身（例如：未经过压缩和修改）。除非特别指明，这个标记始终可以被接受。
- `*` 匹配其他任意未在该首部字段中列出的编码方式。假如该首部字段不存在的话，这个值是默认值。它并不代表任意算法都支持，而仅仅表示算法之间无优先次序。
- `;q= (qvalues weighting)`  值代表优先顺序，用相对质量价值 表示，又称为权重。 

## Accept-Language
浏览器申明自己接收的语言 语言跟字符集的区别
```
Accept-Language:zh-CN,zh;q=0.9
```

## Connection
- `Connection: keep-alive`  当一个网页打开完成后，客户端和服务器之间用于传输HTTP数据的TCP连接不会关闭，如果客户端再次访问这个服务器上的网页，会继续使用这一条已经建立的连接。
- `Connection: close` 代表一个Request完成后，客户端和服务器之间用于传输HTTP数据的TCP连接会关闭， 当客户端再次发送Request，需要重新建立TCP连接。

## Keep-Alive：
如果浏览器请求保持连接，则该头部表明希望 WEB 服务器保持连接多长时间（秒）。
```
Keep-Alive：300
```

## Host（发送请求时，该报头域是必需的）
请求报头域主要用于指定被请求资源的Internet主机和端口号，它通常从HTTP URL中提取出来的。
```
Host:www.baidu.com
```

## Referer
当浏览器向web服务器发送请求的时候，一般会带上Referer，告诉服务器我是从哪个页面链接过来的，服务器籍此可以获得一些信息用于处理。
```
Referer:https://www.baidu.com/?tn=62095104_8_oem_dg
```

## User-Agent
告诉HTTP服务器， 客户端使用的操作系统和浏览器的名称和版本。
```
User-Agent:Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.110 Safari/537.36
```
## Cache-Control
通用消息头字段，被用于在http请求和响应中，通过指定指令来实现缓存机制。缓存指令是单向的，这意味着在请求中设置的指令，不一定被包含在响应中。

- `Cache-Control:private` 默认为private  响应只能够作为私有的缓存，不能再用户间共享
- `Cache-Control:public` 响应会被缓存，并且在多用户间共享。
- `Cache-Control:must-revalidate`  响应在特定条件下会被重用，以满足接下来的请求，但是它必须到服务器端去验证它是不是仍然是最新的。
- `Cache-Control:no-cache`  响应不会被缓存,而是实时向服务器端请求资源。
- `Cache-Control:max-age=10` 设置缓存最大的有效时间，但是这个参数定义的是时间大小（比如：60）而不是确定的时间点。单位是[秒]。
- `Cache-Control:max-stale=10` 
- `Cache-Control:max-fresh=10` 
- `Cache-Control:no-store` 在任何条件下，响应都不会被缓存，并且不会被写入到客户端的磁盘里，这也是基于安全考虑的某些敏感的响应才会使用这个。
- `Cache-Control:no-transform`
- `Cache-Control:only-if-cached`

## Cookie
Cookie是用来存储一些用户信息以便让服务器辨别用户身份的（大多数需要登录的网站上面会比较常见），比如cookie会存储一些用户的用户名和密码，当用户登录后就会在客户端产生一个cookie来存储相关信息，这样浏览器通过读取cookie的信息去服务器上验证并通过后会判定你是合法用户，从而允许查看相应网页。当然cookie里面的数据不仅仅是上述范围，还有很多信息可以存储是cookie里面，比如sessionid等。

## Range（用于断点续传）
指定第一个字节的位置和最后一个字节的位置。用于告诉服务器自己想取对象的哪部分。
```
Range:bytes=0-5
```

## Authorization
当客户端接收到来自WEB服务器的 WWW-Authenticate 响应时，用该头部来回应自己的身份验证信息给WEB服务器。

# 响应头
Cache-control: must-revalidate
Cache-control: no-cache
Cache-control: no-store
Cache-control: no-transform
Cache-control: public
Cache-control: private
Cache-control: proxy-revalidate
Cache-Control: max-age=<seconds>
Cache-control: s-maxage=<seconds>

https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Accept

https://www.cnblogs.com/lguow/p/10620940.html

https://www.byvoid.com/zhs/blog/http-keep-alive-header

https://juejin.im/post/5c17d3cd5188250d9e604628

https://www.jianshu.com/p/347416aafd3f

https://blog.csdn.net/liyantianmin/article/details/82505634

http://51write.github.io/2014/04/09/keepalive/

